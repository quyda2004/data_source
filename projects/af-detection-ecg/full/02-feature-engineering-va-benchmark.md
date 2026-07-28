# AF Detection ECG — Feature Engineering & Benchmark chi tiết

*(Số liệu về dung lượng dữ liệu, số epoch, và các % kết quả cuối trong tài liệu này mang tính minh họa/tham khảo cho mục đích trình bày, không phải số liệu benchmark chính thức)*

---

## Hướng 1: Device — chi tiết technical

### Dữ liệu
- Tín hiệu ECG liên tục, tần số lấy mẫu **360Hz** (360 điểm/giây).
- **44 người này là tập test/evaluation** để đo độ chính xác nhận diện QRS — tách biệt hoàn toàn với tập dùng để **train** model (tránh data leakage).
- Tập train thực tế là 1 bộ dữ liệu ECG khác, quy mô lớn hơn nhiều (nhiều người/nhiều giờ ghi tín hiệu hơn), dùng để huấn luyện CNN trước khi đem model đi test trên 44 người này.
- Trong 44 người của tập test, chia thêm 22/22 để vừa **tune/validate** vừa **test** cuối cùng, đảm bảo con số 99.92% là đo trên phần hoàn toàn chưa từng dùng để chỉnh model.

### Tiền xử lý (loại bỏ baseline wander)
- Downsample tín hiệu gốc 360Hz xuống còn **50Hz** để tạo ra 1 đường trôi nền (baseline) ước lượng bằng phương pháp linear.
- Chỉ lấy khung **4 giây** vì model linear cần ít thông tin để học đường baseline.
- Các sóng QRS (gợn sóng cao tần) bị model linear coi là **outlier** nên không học theo → khi lấy tín hiệu gốc trừ đi baseline sẽ không làm mất các sóng quan trọng.
- Sau đó **normalize** toàn bộ tín hiệu để đồng nhất biên độ giữa các người.

### Model nhận diện QRS (1-D CNN)
- Gắn nhãn theo R-peak: label **1** cho các điểm trong khoảng **±40ms** quanh R-peak (~11 điểm), còn lại label **0**.
- Mỗi sample lấy quanh R-peak: từ **-100ms đến +300ms** (tổng ~400ms).
- Train 1-D CNN để học phân loại điểm thuộc QRS hay không.

### Hậu xử lý bằng K-means
- Trong cửa sổ **80ms**, nếu có trên **5 điểm** được gán label 1 → lấy điểm trung tâm của cụm đó làm vị trí R-peak chính thức.

### Kết quả nhận diện QRS
- **Accuracy: 99.92%** đo trên tập 22 người test (thuộc bộ 44 người test/eval, hoàn toàn tách biệt với tập train).

### Bước phân loại AF (sau khi có R-peak)
- Bệnh được nhận biết qua việc các khoảng cách giữa các R-peak đều hay không đều → extract **7 feature** dựa trên vị trí R-peak.
- Đưa 7 feature qua các model machine learning truyền thống (ví dụ Random Forest / XGBoost) để phân loại AF / Normal.
- **Kết quả phân loại minh họa: ~96.5% accuracy, F1-score ~95.8%** trên tập test giữ lại.
- Test thêm khả năng chịu nhiễu (robustness) trên các mức SNR khác nhau, kết quả minh họa dao động khoảng **90-98%** tùy mức nhiễu.

---

## Hướng 2: Cloud — chi tiết technical

### Tiền xử lý
- Thử tách nhiễu theo các **dải tần số (frequency band)** khác nhau, vì một số loại nhiễu chỉ tác động ở dải tần cố định.
- Sau đó dùng **median filter** để lọc lại tín hiệu.

### Kiến trúc DeepCNN + BiLSTM
- **DeepCNN 24 lớp convolution**, chia làm 3 block:
  - 8 lớp đầu: filter = 32
  - 8 lớp tiếp theo: filter = 64
  - 8 lớp tiếp theo: filter = 128
  - Mỗi block có 1 lớp pooling (hệ số 4)
  - Thiết kế sâu để model có thể bao quát và học được từng thành phần cấu thành của 1 sóng QRS (nếu học ít lớp quá sẽ không bao quát hết được hình thái sóng)
- **2 lớp BiLSTM** sau CNN:
  - Lớp BiLSTM đầu tiên: học mối quan hệ **cục bộ (local)** — input/output cùng số điểm (ví dụ 141 điểm vào → 141 điểm ra), mỗi điểm output mang thông tin quan hệ với các điểm lân cận trước/sau.
  - Lớp BiLSTM thứ hai: học mối quan hệ **toàn cục (global)** trên toàn bộ chuỗi tín hiệu.

### Quy mô training (minh họa)
- Dữ liệu training quy mô lớn, gộp từ nhiều nguồn ECG public (ước tính hàng triệu điểm tín hiệu sau khi windowing).
- Train khoảng **150-200 epoch**, dùng learning rate scheduler + early stopping để tránh overfit.
- Batch size lớn (256-512) tận dụng được khi chạy trên cloud GPU.

### Kết quả (minh họa)
- **Accuracy phân loại (Normal/AF/Other/Noisy): ~98.3%**.
- Test chéo trên các bộ dataset ECG public quốc tế khác (ngoài tập train) để kiểm tra khả năng tổng quát hóa, đạt **accuracy ~95-97%** tùy bộ dữ liệu.
- Test theo chuẩn quốc tế **AAMI EC57** (chuẩn đánh giá thiết bị/thuật toán phát hiện loạn nhịp tim) — đạt các tiêu chí chuẩn với **Sensitivity ~98.1%, Positive Predictivity ~97.6%**, đáp ứng ngưỡng khuyến nghị của chuẩn này.
