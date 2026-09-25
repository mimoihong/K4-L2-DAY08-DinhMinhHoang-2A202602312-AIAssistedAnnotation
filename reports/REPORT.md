# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đinh Minh Hoàng

Công cụ gán nhãn đã dùng: CVAT (kết hợp Ultralytics YOLO format)

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được trích xuất từ một cảnh quay camera cố định nhìn xuống đường cao tốc vào ban đêm. Vì camera đặt tĩnh tại một vị trí, tốc độ lấy mẫu là 2.5 khung hình/giây nên hai frame liên tiếp chỉ cách nhau 0.4 giây. Trong thực tế, một chiếc ô tô di chuyển qua khung hình mất từ vài giây đến hơn mười giây. 

Nếu ta chia tập train/test ngẫu nhiên (random split), các frame liên tiếp của cùng một chiếc xe sẽ xuất hiện đồng thời ở cả tập huấn luyện và tập kiểm thử. Điều này gây ra hiện tượng rò rỉ dữ liệu (data leakage) nghiêm trọng. Khi đó, mô hình kiểm thử trên chính những chiếc xe, góc chiếu và điều kiện ánh sáng mà nó vừa học, dẫn đến số đo hiệu năng (AP50, Precision, Recall) bị thổi phồng giả tạo (overoptimistic), không phản ánh đúng năng lực tổng quát hóa của mô hình trên dữ liệu mới.

Do đó, tập dữ liệu bắt buộc phải được chia theo trục thời gian (temporal split) với 4 cửa sổ kiểm thử độc lập (ở các mốc giây 20, 60, 100, 140) và được bao quanh bởi các vùng đệm thời gian (buffer ít nhất 4.4 giây). Vùng đệm này đảm bảo mọi xe xuất hiện trong tập test đã hoàn toàn đi ra khỏi khung hình trước hoặc sau khi các frame thuộc tập pool được trích xuất.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 từ `reports/rounds_table.md`:
| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào `outputs/compare_round0.jpg` và các chỉ số kích thước, mô hình khởi đầu lạnh (yolov8n pretrained trên COCO) đạt độ chính xác (Precision) rất cao ở ngưỡng 0.25 (92.5%) nhưng độ phủ (Recall) lại thấp (chỉ 48.9%). 
- Về kích thước: Xe lớn (large) và xe vừa (medium) đạt độ phủ tương đối ổn định (lần lượt 56.1% và 54.7%). Tuy nhiên, đối với xe nhỏ (small - thường ở xa sát đường chân trời hoặc chân cầu), độ phủ cực kỳ thấp chỉ đạt 18.2% (12/66 box). Điều này cho thấy model gốc bỏ sót phần lớn các xe ở xa trong điều kiện thiếu sáng.
- Về trường hợp nhãn tham chiếu: Quan sát `compare_round0.jpg`, có những vùng vệt đèn pha rọi sáng trên mặt đường hoặc cụm đèn đường phản chiếu bị nhãn tham chiếu (ground truth) gắn box hoặc ngược lại mô hình phát hiện đúng một xe tải tối màu nhưng nhãn tham chiếu bỏ sót. Vì nhãn tham chiếu của tập test được tạo tự động bởi mô hình khác và chưa qua rà soát thủ công của con người (human-in-the-loop), nên không thể xem nhãn tham chiếu là chân lý tuyệt đối. Cần rà soát lại ground truth trước khi khẳng định mô hình dự đoán sai.

## 3. Chiến lược chọn mẫu

Học chủ động lựa chọn các khung hình cho con người gán nhãn dựa trên công thức tính điểm tổng hợp:
`score = W_U · U + W_A · A + W_D · D` với các trọng số `W_U = 0.5`, `W_A = 0.3`, `W_D = 0.2`:
- `U` (Uncertainty - Trọng số 0.5): Đo lường độ bất định trung bình của mô hình trên các box dự đoán. Những ảnh có confidence dao động quanh vùng ranh giới quyết định sẽ có điểm U cao, cần con người can thiệp nhất.
- `A` (Ambiguity - Trọng số 0.3): Tỷ lệ số box mơ hồ (box có độ tin cậy thấp hoặc gần ngưỡng phát hiện). Ảnh có nhiều xe bị che khuất hoặc ánh sáng mờ sẽ có A cao.
- `D` (Diversity - Trọng số 0.2): Đo độ phân tán theo thời gian so với các ảnh đã chọn, tránh gom các ảnh quá sát nhau.
- `MIN_GAP_S = 2.0s`: Tham số lọc bắt buộc quy định khoảng cách tối thiểu giữa 2 frame được chọn trong cùng một đợt (lô). Do camera tĩnh, nếu chọn 2 frame cách nhau dưới 2 giây thì cảnh vật và xe cộ hầu như trùng lặp 100%, gây lãng phí ngân sách gán nhãn của con người.

Minh chứng cân nhắc mẫu qua 4 frame:
- `frame_0182.jpg` (Rank 1, Score 0.9591): Có độ bất định rất cao (U=0.9182, A=1.0) với 18 box mơ hồ, được chọn đứng đầu lô.
- `frame_0369.jpg` (Rank 2, Score 0.9324): Lưu lượng xe dày đặc (43 box) và 16 box mơ hồ, mang lại mật độ thông tin học tập rất cao.
- `frame_0099.jpg` (Rank 8, Score 0.9063): Mặc dù rank 8 nhưng được ưu tiên vì nằm ở phân đoạn đầu video (giây 39.6), giúp dàn trải đều bối cảnh thời gian thay vì dồn vào cuối video.
- `frame_0372.jpg` (Rank 6, Score 0.9101): Có điểm score cao hơn nhiều frame khác nhưng bị thuật toán loại bỏ do xuất hiện ở giây 148.8, chỉ cách frame_0369.jpg (giây 147.6) vỏn vẹn 1.2s (< `MIN_GAP_S`). Quyết định này giúp tiết kiệm công sức gán nhãn cho cùng một nhóm phương tiện.

*Lưu ý quan trọng:* Điểm số bất định cao (`score`) chỉ chứng minh mô hình **đang phân vân và thiếu chắc chắn nhất**, chứ **không chứng minh** rằng việc gán nhãn ảnh đó chắc chắn sẽ cải thiện được mô hình. Nếu các box không chắc chắn thực chất là nhiễu ảnh, ánh đèn phản quang hoặc xe quá nhỏ dưới 16px thì việc cố gắng gán nhãn có thể gây mâu thuẫn dữ liệu.

## 4. Các vòng học chủ động (active learning)

Bảng so sánh tổng hợp từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 170 | 0.456 | -0.315 | 1.000 | 0.020 | 0.039 | 0.000 | 0.020 | 0.049 |

Phân tích chi tiết vòng 1:
- **Mức độ chỉnh sửa nhãn** (từ `outputs/round1_diff.md`): Trên 12 ảnh với 169 box gợi ý ban đầu, tổng số box sau khi rà soát là 170 box. Trong đó có 166 box được giữ nguyên (`accepted`), 2 box được kéo chỉnh mép viền cho sát thân xe (`edited`), 1 box vệt đèn pha phản chiếu bị xóa (`deleted`) và 2 box xe bị bỏ sót được thêm mới (`added`). Tỷ lệ chấp nhận đạt 98%.
- **Biến thiên hiệu năng**: AP50 giảm từ 0.771 xuống 0.456 (giảm 0.315). Precision tăng tuyệt đối lên 1.000 (không còn bất kỳ False Positive nào ở conf 0.25), nhưng Recall giảm mạnh từ 0.489 xuống 0.020 (chỉ nhận diện được 8 box xe trên 20 ảnh test). Nhóm xe nhỏ Recall về 0.000, nhóm xe vừa đạt 0.020 và xe lớn đạt 0.049.
- **Quan sát định tính trên ảnh `compare_round1.jpg` so với `compare_round0.jpg`**: Sau khi fine-tune 50 epochs chỉ với 12 ảnh (170 box), mô hình trở nên cực kỳ thận trọng và khắt khe. Nó loại bỏ hoàn toàn các box nghi ngờ (Precision = 100%), nhưng lại dẫn đến hiện tượng quá khắt khe khiến các xe ở cự ly xa hoặc xe bị che khuất không đạt ngưỡng confidence 0.25 để xuất hiện.
- **Đối chiếu với `BLIND_SCAN.md` và `REVIEW_LOG.csv`**:
  - `BLIND_SCAN.md` ghi nhận quan sát độc lập ban đầu bằng mắt trên `frame_0099.jpg`: con người nhìn thấy chiếc xe bị cắt mép ở góc dưới bên phải và các đốm xe mờ xa chân cầu.
  - `REVIEW_LOG.csv` ghi lại hành vi sửa nhãn có chủ đích: bổ sung box xe cắt mép cho `frame_0099.jpg` (`added`), xóa vệt phản chiếu đèn trên `frame_0107.jpg` (`deleted`), và căn chỉnh khung viền thân xe ở `frame_0182.jpg` (`edited`).
  - Kết quả mô hình sau train phản ánh rõ tính nhất quán: việc xóa các box không phải xe giúp model triệt tiêu hoàn toàn dự đoán rác, tuy nhiên lượng dữ liệu 12 ảnh là quá nhỏ (small sample size) dẫn đến hiện tượng catastrophic forgetting (quên một phần trọng số tổng quát của bộ COCO) hoặc bị lệch phân phối confidence.

## 5. Kết luận và giới hạn

So với cold start ban đầu, mô hình sau vòng 1 đạt độ tinh lọc cực cao (Precision = 1.0) nhưng bị suy giảm Recall trên tập kiểm thử 20 ảnh. 

Quyết định cho vòng tiếp theo: Nên tiếp tục thu thập thêm vòng 2 (thêm 12-16 ảnh) nhưng cần điều chỉnh chiến lược:
1. Hai ca còn yếu cần tập trung:
   - Các xe ở cự ly xa có kích thước nhỏ (box dưới 32px) để khôi phục lại Recall cho nhóm xe nhỏ.
   - Các xe ở làn đường có mật độ đèn rọi mạnh gây lóa.
2. Cân nhắc chi phí và nguy cơ: Tiếp tục duy trì `MIN_GAP_S` để tránh gán nhãn trùng lặp các xe trong cùng luồng giao thông.
3. Giới hạn đánh giá:
   - Tập kiểm thử chỉ có 20 ảnh (403 box tham chiếu), bất kỳ sự thay đổi của một vài frame cũng có thể tạo ra dao động lớn về số đo.
   - Việc quy định bỏ qua các box cao dưới 16 pixel là hợp lý vì xe quá xa không thể phân biệt ranh giới thân xe.
   - Nhãn tham chiếu của tập test được sinh tự động bởi AI và chưa được con người thẩm định 100%, do đó việc mAP50 giảm không đồng nghĩa hoàn toàn là mô hình học kém đi trong thực tế, mà một phần do mô hình mới đã loại bỏ các dự đoán không trùng khớp với nhãn tự động của tập test.
4. Biện pháp khắc phục khi AP50 giảm: Cần kiểm tra lại phân phối confidence threshold, giảm bớt learning rate khi fine-tune, và bổ sung thêm dữ liệu đa dạng ở vòng 2 trước khi kết luận về chất lượng thuật toán.
