# Vì sao chọn lô này?

## Năm frame ưu tiên nếu chỉ đủ công rà năm ảnh

Tôi xét 50 dòng đầu của `outputs/selection_round1.csv`, nhưng không lấy máy móc năm dòng có điểm cao nhất vì nhiều frame nằm rất gần nhau trên cùng một video. Danh sách ưu tiên là:

| ưu tiên | frame | hạng trong CSV | thời điểm (giây) | score | lý do |
| ---: | --- | ---: | ---: | ---: | --- |
| 1 | `frame_0182.jpg` | 1 | 72.8 | 0.9591 | Điểm tổng cao nhất; `U=0.9182`, `A=1.0`, có 18 box mập mờ. Ảnh có nhiều xe xa, xe sát nhau và vùng chói nên vừa nhiều thông tin vừa dễ có box sai. |
| 2 | `frame_0369.jpg` | 2 | 147.6 | 0.9324 | `U=0.9315`, 16 box mập mờ trên 43 dự đoán. Đây là đại diện cho cụm giao thông dày ở cuối video; chỉ chọn một frame đại diện để hạn chế gán các ảnh gần trùng. |
| 3 | `frame_0326.jpg` | 4 | 130.4 | 0.9155 | `U=0.9310`, 15 box mập mờ; nhiều xe nhỏ ở xa và xe bị che một phần. Thời điểm khác cụm 147–152 giây. |
| 4 | `frame_0099.jpg` | 8 | 39.6 | 0.9063 | `U=0.9460`, 14 box mập mờ. Quét độc lập thấy 22 xe trong khi pre-label chỉ có 13 box, nên chi phí rà cao nhưng khả năng tìm được bỏ sót cũng cao. |
| 5 | `frame_0270.jpg` | 13 | 108.0 | 0.8878 | `U=0.9089`, 14 box mập mờ và nằm ở một đoạn thời gian riêng, giúp lô năm ảnh đa dạng hơn thay vì chọn thêm ảnh quanh 130 hoặc 150 giây. |

## Ba frame thuộc lô 12 ảnh model chọn

- `frame_0182.jpg` đứng hạng 1. Contact sheet cho thấy giao thông dày, xe gần và xa cùng xuất hiện dưới ánh đèn mạnh. Giá trị `A=1.0` và 18 box mập mờ giải thích vì sao model ưu tiên ảnh này.
- `frame_0369.jpg` đứng hạng 2. Ảnh có nhiều xe ở nhiều kích thước, cả đèn hậu và đèn pha; 43 dự đoán làm chi phí rà lớn nhưng cũng tạo nhiều cơ hội phát hiện box thiếu hoặc lệch.
- `frame_0099.jpg` đứng hạng 8. `U=0.9460` cho thấy các box khó có confidence gần vùng phân vân. Quan sát độc lập trước pre-label đã chỉ ra xe nhỏ xa bên trái và xe mờ bị cắt ở mép phải là hai vị trí dễ sai.

## Một frame điểm cao nhưng không chọn

`frame_0372.jpg` đứng hạng 6, score 0.9101 tại giây 148.8, nhưng không được chọn. Nó chỉ cách `frame_0369.jpg` ở giây 147.6 đúng 1.2 giây, nhỏ hơn `MIN_GAP_S=2.0`. Với camera cố định, hai ảnh có khả năng chứa nhiều xe và bố cục gần trùng; gán cả hai sẽ tốn công mà thông tin mới ít. Nếu chỉ có ngân sách năm ảnh, tôi giữ `frame_0369.jpg` có score cao hơn và bỏ `frame_0372.jpg`.

## Điều phép chọn này chưa chứng minh

Điểm cao chỉ cho biết model hiện tại phân vân, có nhiều box confidence trung bình hoặc ảnh cách xa dữ liệu đã gán. Nó không chứng minh nhãn AI sai, ảnh đại diện cho toàn bộ miền dữ liệu, hay việc gán ảnh đó chắc chắn làm AP50 tăng. Độ chói, mờ, xe quá nhỏ, box tham chiếu chưa được người rà và các frame gần trùng đều có thể tạo điểm bất định cao mà không mang lại tín hiệu huấn luyện hữu ích tương ứng.
