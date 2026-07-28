# AF Detection ECG — Bối cảnh & Kiến trúc 2 hướng Device vs Cloud

## Bối cảnh

- Được công ty giao nghiên cứu theo 2 hướng song song: **Device** (chạy trên thiết bị, ưu tiên nhanh & nhẹ) và **Cloud** (chạy trên server, có thể dùng model lớn hơn, tích hợp kết quả từ các paper đã tham khảo trước đó).
- Bản chất bài toán: rung nhĩ (AF - Atrial Fibrillation) được nhận biết qua sự **loạn nhịp giữa các sóng QRS** — người bình thường nhịp đều, người bệnh nhịp không đều theo từng event.

*(Số liệu về dung lượng dữ liệu, số epoch, và các % kết quả cuối trong tài liệu này mang tính minh họa/tham khảo cho mục đích trình bày, không phải số liệu benchmark chính thức)*

---

## Hướng 1: Device (ưu tiên tốc độ infer & model nhỏ) — tổng quan

### Định hướng
Vì ưu tiên infer nhanh và model nhỏ để chạy được trên thiết bị, chọn hướng **machine learning truyền thống** thay vì deep learning nặng. Tuy nhiên cần qua 1 bước nhận diện QRS trước khi extract feature.

### Pipeline high-level:
1. Tiền xử lý loại baseline wander.
2. 1-D CNN nhận diện QRS + K-means hậu xử lý → có vị trí R-peak.
3. Extract 7 feature dựa trên khoảng cách các R-peak.
4. Đưa 7 feature qua ML truyền thống (Random Forest / XGBoost) để phân loại AF/Normal.

**Kết quả tổng thể:** QRS detection accuracy 99.92%, AF classification ~96.5% accuracy, F1 ~95.8%. Robust trên các mức SNR khác nhau ~90-98%.

Chi tiết technical + hyperparameter + kết quả từng bước xem trong `02-feature-engineering-va-benchmark.md`.

---

## Hướng 2: Cloud (tích hợp kết quả các paper tham khảo) — tổng quan

### Định hướng
Tận dụng compute cloud để dùng deep learning end-to-end, học được đầy đủ hình thái sóng thay vì chỉ khoảng cách R-peak.

### Pipeline high-level:
1. Tiền xử lý tách nhiễu theo dải tần số + median filter.
2. **DeepCNN 24 lớp** + **2 lớp BiLSTM** (lớp 1 học local, lớp 2 học global).
3. End-to-end phân loại 4 class: Normal / AF / Other / Noisy.

**Kết quả tổng thể:** Accuracy ~98.3% trên tập test, cross-dataset ~95-97%, đạt chuẩn quốc tế **AAMI EC57** với Sensitivity ~98.1%, Positive Predictivity ~97.6%.

Chi tiết technical + hyperparameter + benchmark xem trong `02-feature-engineering-va-benchmark.md`.

---

## So sánh nhanh 2 hướng

| Tiêu chí | Device | Cloud |
|---|---|---|
| Mục tiêu | Nhanh, nhẹ, chạy on-device | Độ chính xác cao, tận dụng compute lớn |
| Cách tiếp cận | CNN nhận diện QRS + ML truyền thống trên feature | End-to-end DeepCNN + BiLSTM |
| Input | R-peak feature (7 feature) | Toàn bộ waveform |
| Ưu điểm | Tốc độ infer nhanh, model nhỏ | Học được hình thái sóng đầy đủ hơn, độ chính xác cao hơn |
| Đánh đổi | Có thể bỏ sót thông tin hình thái sóng phức tạp | Nặng hơn, cần compute lớn, không phù hợp on-device |

---

## Tham khảo

- **Šarlija et al. (ISPA 2017)** — CNN-based QRS detection (nền cho hướng Device).
- **Cheng et al. (2021)** — DCNN + BiLSTM, classify ECG thành Normal/AF/Other/Noisy (nền cho hướng Cloud).
- **Chuẩn AAMI EC57** — chuẩn đánh giá thiết bị/thuật toán phát hiện loạn nhịp tim (dùng để chấm hướng Cloud).
