# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà năm ảnh, tôi ưu tiên 5 frame sau:
1. **frame_0182.jpg** (Hạng 1, t = 72.8s, Score = 0.9591, U = 0.9182, A = 1.0, D = 1.0): Điểm cao nhất toàn tập dữ liệu, có tới 18/28 box mơ hồ (A=1.0) và độ bất định rất cao.
2. **frame_0369.jpg** (Hạng 2, t = 147.6s, Score = 0.9324, U = 0.9315, A = 0.8889, D = 1.0): Lưu lượng xe dày đặc (43 box) với mật độ box không chắc chắn cao (16 box mơ hồ), đem lại lượng thông tin lớn.
3. **frame_0380.jpg** (Hạng 3, t = 152.0s, Score = 0.9170, U = 0.9340, A = 0.8333, D = 1.0): Cách frame_0369.jpg 4.4 giây (lớn hơn MIN_GAP_S = 2.0s), bối cảnh xe đã thay đổi đáng kể, độ bất định cao (U=0.9340).
4. **frame_0326.jpg** (Hạng 4, t = 130.4s, Score = 0.9155, U = 0.9310, A = 0.8333, D = 1.0): Xuất hiện ở khoảng thời gian khác, có 39 xe và độ bất định cao.
5. **frame_0099.jpg** (Hạng 8, t = 39.6s, Score = 0.9063, U = 0.9460, A = 0.7778, D = 1.0): Dù đứng hạng 8 nhưng tôi chọn thay vì frame_0331.jpg (hạng 5, t = 132.4s) vì frame_0331 chỉ cách frame_0326 đúng 2.0s nên dễ bị trùng lặp xe di chuyển trên cùng đoạn đường. frame_0099.jpg nằm ở đầu video (giây 39.6), đa dạng hóa phân phối dữ liệu và có độ bất định rất cao (U = 0.9460).

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
- **frame_0182.jpg**: Score = 0.9591 (Rank 1), model dự đoán 28 box nhưng có đến 18 box rơi vào vùng tin cậy thấp/mơ hồ (conf gần ngưỡng 0.25). Trên ảnh contact sheet, các xe di chuyển ở làn xa và góc khuất ánh sáng bị mờ nhạt.
- **frame_0099.jpg**: Score = 0.9063 (Rank 8), U = 0.9460. Trên contact sheet cho thấy tình trạng vệt đèn phản chiếu trên mặt đường ướt bị model dự đoán nhầm hoặc xe sát lề phải bị cắt nửa thân xe.
- **frame_0107.jpg**: Score = 0.8876 (Rank 14), t = 42.8s. Cách frame_0099 đúng 3.2s (đảm bảo đa dạng), model gặp khó khăn với các cụm xe tải và vệt đèn pha rọi ngược.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- **frame_0372.jpg** (Hạng 6, Score = 0.9101, U = 0.9202): Mặc dù có điểm số xếp hạng cao thứ 6 (cao hơn nhiều ảnh được chọn như frame_0099, frame_0107), thuật toán loại bỏ không chọn vào lô vì nó xuất hiện ở t = 148.8s, chỉ cách frame_0369.jpg (t = 147.6s) vỏn vẹn 1.2 giây (nhỏ hơn `MIN_GAP_S = 2.0s`). Việc loại bỏ frame này là hoàn toàn chính xác để tránh trùng lặp dữ liệu và lãng phí công rà nhãn cho cùng những chiếc xe đang di chuyển.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Phép chọn dựa vào hàm điểm kết hợp độ bất định (`U`), số box mơ hồ (`A`) và độ phân tán thời gian (`D`). Điểm số cao chỉ phản ánh rằng **mô hình hiện tại đang phân vân và thiếu tự tin nhất** trên những khung hình đó, chứ **hoàn toàn chưa chứng minh** rằng việc gán nhãn bổ sung cho 12 ảnh này chắc chắn sẽ làm tăng mAP50 trên tập kiểm thử. Nếu các box không chắc chắn rơi vào các trường hợp nhiễu (vệt đèn, đốm sáng xa dưới 16px) hoặc việc gán nhãn không nhất quán, mô hình thậm chí có thể học sai quy luật và làm giảm hiệu năng.
