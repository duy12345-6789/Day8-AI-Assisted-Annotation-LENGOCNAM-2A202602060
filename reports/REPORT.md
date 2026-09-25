# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: LENGOCNAM

Công cụ gán nhãn đã dùng: CVAT Docker v2.76.0, định dạng Ultralytics YOLO Detection 1.0

## 1. Dữ liệu và cách chia tập

Video là một cảnh quay liên tục từ camera cố định và được lấy mẫu 2.5 frame/giây, nên hai ảnh liên tiếp chỉ cách nhau 0.4 giây và thường chứa cùng một chiếc xe. Nếu chia ngẫu nhiên, cùng xe và gần như cùng nền có thể xuất hiện ở cả train và test. Đây là rò rỉ dữ liệu và sẽ làm số đo test cao hơn khả năng tổng quát hóa thực tế.

Dữ liệu vì vậy được chia theo thời gian: 20 ảnh test nằm trong bốn đoạn quanh giây 20, 60, 100 và 140; 112 ảnh vùng đệm bị loại; 268 ảnh còn lại tạo pool. Ảnh pool gần test nhất vẫn cách 4.4 giây. Cách chia này giảm khả năng cùng một xe xuất hiện ở cả hai phía, dù chưa loại bỏ hoàn toàn sự giống nhau về góc máy và điều kiện ban đêm.

## 2. Mô hình khởi đầu lạnh (cold start)

| vòng | model | ảnh train | box train | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Cold start có precision cao nhưng recall thấp: tại confidence 0.25 có 197 TP, 16 FP và 206 FN. `compare_round0.jpg` cho thấy model thường bỏ sót xe nhỏ/xa gần đường chân trời, xe tối hoặc bị che, và một số xe chỉ hiện qua cụm đèn. Recall small chỉ 0.182, thấp hơn rõ rệt medium 0.547 và large 0.561, nên kích thước nhỏ là điểm yếu chính.

Không phải mọi khác biệt với reference đều chắc chắn là lỗi model. Ví dụ các xe rất xa trong `frame_0350` chỉ còn cụm đèn nhỏ có thể khó xác định ranh giới thân xe; xe bị cắt ở mép ảnh cũng dễ có box khác nhau. Nhãn test do một model khác tạo và chưa được người rà, nên cần xem ảnh gốc và guideline trước khi coi một FP/FN là chân lý.

## 3. Chiến lược chọn mẫu

Công thức `score = 0.5·U + 0.3·A + 0.2·D` kết hợp ba tín hiệu. `U` là trung bình năm độ bất định box lớn nhất, cao nhất khi confidence gần 0.5. `A` là số box có confidence từ 0.15 đến dưới 0.50, được chuẩn hóa theo pool. `D` là khoảng cách thời gian đến frame đã gán gần nhất, chặn ở 10 giây và chuẩn hóa; ở vòng đầu chưa có frame đã gán nên `D=1` cho mọi ảnh. `MIN_GAP_S=2.0` ngăn hai frame quá gần nhau cùng vào lô, vì camera cố định làm chúng gần trùng và chi phí rà không tạo nhiều thông tin mới.

Ba ví dụ chi tiết trong `reports/SELECTION.md` là `frame_0182.jpg` (hạng 1, score 0.9591, 18 box mập mờ), `frame_0369.jpg` (hạng 2, score 0.9324, 16 box mập mờ) và `frame_0099.jpg` (hạng 8, score 0.9063, `U=0.9460`). `frame_0372.jpg` có score 0.9101 nhưng cách `frame_0369.jpg` chỉ 1.2 giây nên bị bỏ để tránh ảnh gần trùng. Nếu chỉ đủ rà năm ảnh, tôi ưu tiên các thời điểm 39.6, 72.8, 108.0, 130.4 và 147.6 giây để cân bằng bất định với chi phí và đa dạng thời gian.

Điểm bất định không chứng minh ảnh sẽ cải thiện mô hình. Nó có thể tăng vì chói, mờ, xe quá nhỏ, box khó định nghĩa hoặc model đang sai có hệ thống; ảnh gần trùng còn có thể lặp lại cùng lỗi mà không thêm thông tin mới.

## 4. Các vòng học chủ động (active learning)

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 329 | 0.470 | -0.302 | 0.974 | 0.184 | 0.309 | 0.000 | 0.193 | 0.415 |

Ở vòng 1, model ban đầu đề xuất 169 box. Tôi giữ 156, chỉnh 8, xóa 5 và thêm 165; nhãn cuối có 329 box. `BLIND_SCAN.md` là quan sát độc lập trước khi thấy nhãn: ở `frame_0099.jpg` tôi đếm 22 xe và chỉ ra xe nhỏ xa bên trái cùng xe mờ bị cắt bên phải. Sau đó `round1_diff.md` cho thấy pre-label của frame này chỉ có 13 box, còn nhãn cuối có 23 box: 8 giữ, 4 chỉnh, 1 xóa và 11 thêm. `REVIEW_LOG.csv` ghi các lỗi pre-label cụ thể như hai xe giữa ảnh bị bỏ sót, xe xa/che khuất và box có tỷ lệ sai. Đây là bằng chứng về chất lượng nhãn gợi ý và phần sửa của người rà, chưa phải kết quả model sau train.

Sau fine-tune, AP50 giảm 0.3018, recall giảm từ 0.4888 xuống 0.1836 và F1 giảm từ 0.6396 xuống 0.3090. Precision tăng từ 0.9249 lên 0.9737 vì FP giảm từ 16 xuống 2, nhưng TP giảm từ 197 xuống 74 và FN tăng từ 206 lên 329. Mọi nhóm kích thước đều xấu đi: recall small 0.1818 xuống 0.0000, medium 0.5473 xuống 0.1926, large 0.5610 xuống 0.4146.

`compare_round1.jpg` cho thấy thay đổi này trực tiếp. Ở `frame_0150`, cold start có TP=10, FP=2, FN=10; vòng 1 chỉ còn TP=3, FP=1 và FN=17. Ở `frame_0050`, TP giảm 11 xuống 6 trong khi FP giảm 2 xuống 0. Model sau train thận trọng hơn nhưng bỏ sót nhiều xe hơn, nên không thể gọi đây là cải thiện dù precision tăng.

Một ca khó theo guideline là xe bị cắt ở mép ảnh trong `frame_0182.jpg`: vẫn phải gán nhãn nhưng chỉ ôm phần xe nằm trong ảnh. Xe xa chỉ còn hai chấm đèn và box cao dưới khoảng 16 px có thể gán hoặc không; cần áp dụng nhất quán. Điều này khác với lỗi pre-label và cũng khác với việc model sau fine-tune không phát hiện xe trên test.

## 5. Kết luận và giới hạn

Vòng 1 kém cold start rõ rệt về AP50, recall và F1, dù precision tăng. Tôi dừng sau vòng bắt buộc thay vì train ngay vòng 2, vì lô 12 ảnh nhỏ và mức giảm lớn cho thấy cần QC nhãn/cấu hình trước. Hai ca đáng ưu tiên nếu tiếp tục là: (1) các xe nhỏ/xa gần đường chân trời như cụm xe bị bỏ sót trong `frame_0150` và `frame_0350` của ảnh so sánh; (2) xe bị che, mờ hoặc cắt ở mép như các ca đã gặp trong `frame_0099` và `frame_0182`. Hai test frame chỉ minh họa kiểu lỗi; vòng sau phải tìm ca tương tự trong pool, không đưa ảnh test vào train. Ca thứ nhất tốn công vì phải quét toàn ảnh và quyết định nhất quán với luật bỏ qua box dưới khoảng 16 px; ca thứ hai tốn công xác định phần thân thực sự nhìn thấy và tránh ôm vùng chói. Khi chọn pool cho vòng sau, cần lấy frame đại diện ở các thời điểm khác nhau và tránh ảnh cách nhau dưới 2 giây.

Tập test chỉ có 20 ảnh, 14 box quá nhỏ bị bỏ qua và nhãn tham chiếu do model tạo chưa được người rà. Vì vậy số đo phản ánh mức khớp với reference này, không phải chất lượng thực địa tuyệt đối. Tuy nhiên mức giảm AP50 0.302 lớn hơn nhiều ngưỡng dao động nhỏ khoảng 0.01, nên vẫn là tín hiệu cần điều tra.

Trước khi train thêm, tôi sẽ rà lại tính nhất quán của 329 box: xe xa, xe bị che/cắt mép, vùng phản chiếu và vệt đèn; kiểm tra có box quá rộng hoặc nhãn quá dày so với định nghĩa reference; xác nhận đúng 12 ảnh pool và không có test leakage; rồi kiểm tra checkpoint, số epoch, ngưỡng confidence và ảnh so sánh theo từng frame. Chỉ sau khi QC mới quyết định sửa lô hiện tại, bổ sung một lô đa dạng hơn hoặc điều chỉnh huấn luyện.
