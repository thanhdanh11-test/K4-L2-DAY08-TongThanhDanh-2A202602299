# Vì sao chọn lô này?

Số liệu lấy từ `outputs/selection_round1.csv` và `outputs/selection_round1.jpg`. Vòng 1 chưa có ảnh nào được gán nhãn nên D = 1 với mọi ảnh, thứ hạng chỉ phụ thuộc U và A.

## Top 5 nếu chỉ rà được 5 ảnh

| frame | rank | score | t (s) | lý do |
| --- | ---: | ---: | ---: | --- |
| frame_0182 | 1 | 0.959 | 72.8 | Điểm cao nhất, nhiều box mập mờ nhất pool (18) |
| frame_0369 | 2 | 0.932 | 147.6 | 43 box ở conf ≥ 0.05 nhưng chỉ 14 box qua 0.25 |
| frame_0326 | 4 | 0.916 | 130.4 | U cao, đoạn đông xe |
| frame_0099 | 8 | 0.906 | 39.6 | U cao nhất top 10, là ảnh duy nhất trong top 10 ở đoạn đầu video |
| frame_0227 | 11 | 0.892 | 90.8 | Lấp khoảng trống từ 75 đến 125 s |

Mình không lấy nguyên 5 dòng đầu vì có ảnh gần trùng. frame_0331 (rank 5) chỉ cách frame_0326 2 giây, gần như cùng một đoàn xe. frame_0380 (rank 3) cách frame_0369 4.4 giây. Camera đứng yên nên hai ảnh sát nhau thì gán cả hai cũng không được thêm bao nhiêu. Trong top 50 không có ảnh nào model dự đoán 0 box.

## Ba ảnh trong lô 12

- **frame_0182** (rank 1, A = 1.0): model đề xuất 13 box, sau khi sửa có 26 box, trong đó mình thêm 14 và chỉnh 6. Box ở ảnh này vừa thiếu vừa lệch.
- **frame_0369** (rank 2): model đề xuất 14 box, sau khi sửa có 40 box, thêm 27, nhiều nhất lô. Model thấy được nhiều xe nhưng conf thấp nên bị ngưỡng 0.25 lọc mất.
- **frame_0392** (rank 15): vào lô nhờ U = 0.975, cao nhất top 50, dù A chỉ 0.67. Nó có chỗ vì các ảnh rank 6, 9, 12 bị loại do quá gần ảnh đã chọn. Sau khi sửa từ 13 lên 29 box.

## Ảnh điểm cao nhưng không được chọn

frame_0372 (rank 6, score 0.910) chỉ cách frame_0369 1.2 giây, dưới `MIN_GAP_S` = 2 giây, nên bị bỏ. Tương tự với frame_0368 (rank 9) và frame_0330 (rank 12). Mỗi ảnh tốn khoảng 40 box để rà nhưng gần như lặp lại ảnh đã có.

## Phép chọn này chưa chứng minh gì

U chỉ cho biết model đang phân vân, chứ không cho biết model sai. Xe bị bỏ hẳn (conf < 0.05) thì không làm U tăng. Cũng chưa có vòng chọn ngẫu nhiên để so sánh, nên chưa nói được cách chọn này tốt hơn chọn bừa.
