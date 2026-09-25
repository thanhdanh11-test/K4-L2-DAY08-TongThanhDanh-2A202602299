# Quét độc lập trước khi xem pre-label

Frame: frame\_0099.jpg

Số xe nhìn thấy bằng mắt: 26 xe nhìn thấy , 12 xe đi từ chiều từ trên xuống dưới bức ảnh, 14 xe đi chiều ngược lại từ phía dưới bên phải bức ảnh lên phía trên bức ảnh 

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:   

- vị trí phía xa nằm ở phía trên bên trái các xe hiển thị rất nhỏ nên AI sẽ dễ bỏ qua.
- vị trí sát địa hình bên tay trái các xe chỉ lộ 1 phần đèn xe  nên AI có thể bỏ qua nó không phải là car.
- khả năng vẽ sai vùng cuồng sáng phía trước các xe.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.