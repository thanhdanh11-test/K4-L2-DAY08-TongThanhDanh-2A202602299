# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Tống Thành Danh

Công cụ gán nhãn đã dùng: CVAT v2.76.0 (Docker local), định dạng Ultralytics YOLO Detection 1.0

## 1. Dữ liệu và cách chia tập

Video quay liên tục bằng camera đứng yên, lấy 2.5 frame/giây, nên hai frame liền nhau gần như giống hệt nhau. Nếu chia ngẫu nhiên, gần như ảnh test nào cũng có một ảnh train cách nó 0.4 giây với cùng chiếc xe ở cùng chỗ. Model sẽ được chấm trên xe nó đã thấy, nên số đo test sẽ bị cao hơn thực tế. Chia theo thời gian và bỏ vùng đệm ±4 giây giúp tránh rò rỉ dữ liệu như vậy: ảnh pool gần test nhất vẫn cách 4.4 giây.

## 2. Mô hình khởi đầu lạnh (cold start)

| vòng | ảnh train | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 0 | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Model đoán ít nhưng đoán đúng: precision 0.925, nhưng recall chỉ 0.489 (FN 206 so với TP 197). Xe nhỏ ở xa bị bỏ nhiều nhất (recall 0.182). Mình bất ngờ là xe to gần camera cũng chỉ đạt 0.561. Trong `compare_round0.jpg`, ở frame_0250 hai xe lớn ở góc dưới trái bị bỏ sót, có lẽ vì đèn pha chói quá. Box thừa chủ yếu là box gộp hai xe dính nhau.

Ca nên rà lại nhãn tham chiếu: frame_0150, cụm xe bên phải. Nhãn tham chiếu có hai box chồng lên nhau. Nhãn này cũng do model tạo, nên có thể chính nó đã tách sai một xe thành hai.

## 3. Chiến lược chọn mẫu

`score = 0.5·U + 0.3·A + 0.2·D`. U là mức phân vân trung bình của 5 box khó nhất trong ảnh (cao nhất khi conf ≈ 0.5). A là số box có conf trong khoảng 0.15–0.5, so với ảnh có nhiều box như vậy nhất. D là khoảng cách thời gian tới ảnh đã gán gần nhất. Vòng 1 thì D = 1 với mọi ảnh, nên chỉ còn `MIN_GAP_S` = 2 giây ngăn việc chọn hai ảnh sát nhau.

Ví dụ trong `SELECTION.md`: frame_0182 có A cao nhất và đúng là phải sửa nhiều (thêm 14 box). frame_0369 có 43 box ở conf thấp nhưng chỉ 14 box qua ngưỡng 0.25, mình phải thêm 27 box. frame_0392 vào lô nhờ U cao. frame_0372 (rank 6) bị loại vì cách frame_0369 1.2 giây.

Điểm bất định không chứng minh ảnh đó sẽ giúp model tốt lên. Nó chỉ cho biết model đang phân vân. Xe mà model không thấy chút nào thì không làm tăng điểm này.

## 4. Các vòng học chủ động (active learning)

| vòng | ảnh train | box train | AP50 | Δ AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | 12 | 356 | 0.376 | -0.396 | 1.000 | 0.087 | 0.160 | 0.000 | 0.064 | 0.390 |

Sửa nhãn vòng 1 (`outputs/round1_diff.md`): model đề xuất 169 box, sau khi sửa có 356 box. Giữ 128, chỉnh 24, xóa 17, thêm 204. Box model đưa ra phần lớn dùng được, nhưng hơn một nửa số xe là mình phải tự thêm.

Sau fine-tune, kết quả tệ đi rõ rệt. AP50 giảm từ 0.771 xuống 0.376. Recall giảm từ 0.489 xuống 0.087 (TP 35, FN 368). Precision lên 1.000 chỉ vì model gần như không đoán box nào: FP = 0. Nhóm xe nào cũng giảm: small 0.182 → 0, medium 0.547 → 0.064, large 0.561 → 0.390. Xe lớn giảm ít nhất.

Ví dụ trong `compare_round1.jpg`: ở frame_0050, cold start đúng 11 xe, còn vòng 1 chỉ đúng 2 xe, bỏ sót 16. Box vàng (bỏ sót) nằm đúng những chỗ trước đó model đã bắt được. Ở frame_0250, số box thừa giảm từ 2 xuống 0, nhưng số xe đúng cũng giảm từ 6 xuống 3.

Mình nghĩ nguyên nhân là **model mất tự tin**, không phải học sai vị trí. Bằng chứng trong `selection_round2.csv`: ở ngưỡng conf ≥ 0.05, mỗi ảnh pool giờ chỉ còn 0–7 box, trong khi trước đó là 28–47 box, và frame_0022 không có box nào. AP50 vẫn còn 0.376 trong khi recall ở conf 0.25 chỉ 0.087, nghĩa là model vẫn xếp hạng được một phần xe nhưng conf rất thấp. Lý do có thể kiểm tra: fine-tune với 1 class nên phần head phân loại COCO được khởi tạo lại. Chỉ 12 ảnh gần giống nhau trong 50 epoch thì không đủ để head mới học lại. Mình chưa kiểm chứng giả thuyết này.

Quan sát độc lập (`BLIND_SCAN.md`, frame_0099): mình ghi 26 xe, dự đoán AI sẽ bỏ sót xe nhỏ ở xa phía trên bên trái, xe sát mép trái chỉ lộ đèn, và vẽ lệch do quầng sáng đèn pha. Ảnh này model đề xuất 13 box, mình sửa thành 26 box (thêm 13, chỉnh 6), nên dự đoán "bỏ sót" là đúng. Lưu ý: file này đã bị sửa sau khi khóa nên `blind_lock.json` không còn khớp.

Một số ca trong `REVIEW_LOG.csv`: thêm một xe tối ở mép trên trái frame_0099; chỉnh box xe làn trái frame_0107 bị kéo xuống phần bóng. Ca frame_0270 mình ghi là xóa box trùng, nhưng `round1_diff.json` lại ghi ảnh này xóa 0 box. Có thể box trùng được tính là "edited" khi so IoU; mình chưa kiểm lại được.

Ca khó: xe ở xa chỉ thấy cặp đèn. Mình chọn vẫn vẽ những xe này. Trong `labels/round1/` có 32/356 box cao dưới 16 px. Guideline cho phép vẽ hoặc bỏ, nhưng những box này không được tính khi chấm test.

## 5. Kết luận và giới hạn

Vòng 1 kém hơn cold start ở mọi chỉ số, trừ precision. Mình **dừng, không gán nhãn vòng 2 ngay**. Lô vòng 2 được chọn bởi chính model đã hỏng: nó chọn frame_0022 chỉ vì ảnh này không có box nào, và pre-label chỉ có 0–7 box mỗi ảnh. Rà theo lô này sẽ tốn công gần như gán nhãn từ đầu. Nên sửa cách train trước: giữ head COCO hoặc đóng băng backbone, giảm learning rate, hoặc train thêm cả ảnh đã có nhãn tốt. Sau đó chạy lại để xem AP50 có quay về mức khoảng 0.77 không.

Hai ca cho vòng sau:
1. Xe cỡ vừa ở giữa đường như frame_0050 (test): cold start bắt được nhưng vòng 1 bỏ sót. Đây là dấu hiệu rõ nhất của việc model mất tự tin, nên dùng để kiểm tra lần train sau.
2. frame_0372 (pool, rank 6 ở vòng 1): chỉ nên lấy một ảnh trong cụm này vì nó gần frame_0369 đã gán. Mình ưu tiên ảnh ở đoạn 0–35 giây, là đoạn chưa có ảnh nào được gán.

Giới hạn: test chỉ có 20 ảnh, nhưng mức giảm 0.396 AP50 quá lớn để chỉ là nhiễu. Nhãn test do model tạo, có thể có sai. Xe cao dưới 16 px không được tính, nên 32 box xe nhỏ mình vẽ không ảnh hưởng số đo. Nhãn train chỉ có mình rà, không ai kiểm lại.

Trước khi train thêm, mình sẽ kiểm tra: (1) `labels/round1/` có đúng 12 ảnh pool, class 0 và không lẫn ảnh test; (2) vài ảnh train có box đúng chỗ không; (3) biểu đồ loss của Colab xem model có hội tụ không; (4) phân bố conf của dự đoán trên test, để xác nhận model mất tự tin chứ không phải đoán sai vị trí.
