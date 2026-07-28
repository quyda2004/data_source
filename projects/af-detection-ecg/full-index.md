# AF Detection ECG — Full Index

| File | Nội dung | Preview |
|---|---|---|
| `01-device-vs-cloud-architecture.md` | Bối cảnh AF + tổng quan 2 hướng Device/Cloud + so sánh trade-off + paper tham khảo | Loạn nhịp giữa các sóng QRS; Device ưu tiên nhanh/nhẹ; Cloud dùng compute lớn; Šarlija et al. 2017 + Cheng et al. 2021 |
| `02-feature-engineering-va-benchmark.md` | Chi tiết technical: tiền xử lý baseline wander, 1-D CNN QRS + K-means + 7 feature (Device), DeepCNN 24 lớp + BiLSTM (Cloud) + benchmark AAMI EC57 | Downsample 360→50Hz; label ±40ms; QRS accuracy 99.92%; 3 block × 8 conv filter 32/64/128; 4-class Normal/AF/Other/Noisy accuracy 98.3%; Sensitivity 98.1% |
