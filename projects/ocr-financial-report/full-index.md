# OCR 9000 Báo cáo tài chính — Full Index

| File | Nội dung | Preview |
|---|---|---|
| `01-pipeline-ocr-va-consensus.md` | Pipeline OCR (PDF → ResNet-18 fix hướng → DeepSeek OCR → extract with Evidence+Confidence) + phương pháp consensus 3 LLM | 9000 file nội bộ; 500k→20k+ ảnh sai hướng; sinh 2 bộ dữ liệu con dựa trên keyword; GPT-4o + Claude Haiku 4.5 + Claude 3.5 phải trùng tuyệt đối; kết quả 97.3% / 98.6% |
| `02-so-sanh-mo-hinh.md` | Format dữ liệu chuẩn OCR evaluation + so sánh OCR truyền thống vs DeepSeek vs Qwen2.5-VL + metric đánh giá bảng (TEDS/GriTS/Cell-level/Numeric) | dataset/images/ground_truth/predictions; bbox+text+region_type; task-based evaluation né chấm cấu trúc bảng thô |
