# OCR 9000 — Format Dữ Liệu Chuẩn & So Sánh Model + Metric Đánh Giá Bảng

## 1. Kiến trúc/định dạng dữ liệu chuẩn khi đánh giá model OCR (tham khảo)

Khi đánh giá OCR cho một bài toán cụ thể, format dữ liệu chuẩn thường có cấu trúc dạng sau:

```
dataset/
├── images/                  # Ảnh gốc (đã convert từ PDF hoặc scan)
│   ├── doc_0001_page_01.png
│   └── ...
├── ground_truth/            # Nhãn chuẩn (annotate thủ công hoặc bán tự động)
│   ├── doc_0001_page_01.json   # bounding box + text + label vùng (table/paragraph/title)
│   └── ...
├── predictions/              # Output của model OCR cần đánh giá
│   └── doc_0001_page_01.json
└── metadata.csv              # Thông tin file: nguồn, loại tài liệu, độ khó, ngôn ngữ...
```

**Format 1 record ground truth (JSON) thường gồm:**
- `bbox`: tọa độ vùng (x, y, width, height)
- `text`: nội dung text đúng trong vùng đó
- `region_type`: loại vùng (table, paragraph, title, header, footer...)
- `reading_order`: thứ tự đọc của vùng đó trong trang
- `confidence` (optional): độ tin cậy khi gán nhãn (nếu gán bán tự động)

Cấu trúc này giúp tách riêng việc đánh giá theo từng tiêu chí (CER/WER theo `text`, table structure theo `bbox` + `region_type`, reading order theo `reading_order`) thay vì gộp chung một điểm số duy nhất — phù hợp với cách tiếp cận task-based evaluation.

---

## 2. So sánh minh họa: OCR truyền thống vs DeepSeek OCR vs Qwen2.5-VL

*(Số liệu dưới đây mang tính minh họa/tham khảo, không phải benchmark chính thức)*

| Tiêu chí | OCR truyền thống (Tesseract/PaddleOCR) | DeepSeek OCR | Qwen2.5-VL (VLM) |
|---|---|---|---|
| CER (Character Error Rate) | ~8-12% | ~2-3% | ~2-4% |
| Table structure accuracy | ~65-70% | ~88-90% | ~85-90% |
| Reading order (multi-column) | Yếu, hay lẫn thứ tự cột | Tốt | Tốt |
| Xử lý ảnh xoay/ngược | Cần tiền xử lý riêng (không tự nhận diện) | Có hỗ trợ nhưng vẫn nên tiền xử lý | Nhận diện tốt hơn nhờ context hiểu ảnh tổng thể |
| Hiểu ngữ cảnh/layout phức tạp | Không (chỉ nhận diện ký tự thuần) | Có một phần | Có (VLM hiểu ảnh + text cùng lúc) |
| Tốc độ xử lý | Nhanh, nhẹ | Trung bình | Chậm hơn (model lớn hơn) |
| Chi phí compute | Thấp | Trung bình | Cao |

---

## 3. Các tiêu chí đánh giá riêng cho bảng (table evaluation)

Ngoài các tiêu chí chung ở trên, đánh giá bảng (table) trong OCR có các metric riêng của thế giới:

### 3.1. TEDS (Tree Edit Distance-based Similarity) — metric phổ biến nhất hiện nay
- Coi bảng như 1 cây HTML (row, col, cell, merge...), tính khoảng cách chỉnh sửa (edit distance) giữa cây dự đoán và cây ground truth.
- 2 biến thể: **TEDS-Struct** (chỉ tính đúng cấu trúc hàng/cột, không quan tâm nội dung) và **TEDS** (tính cả cấu trúc lẫn nội dung text trong ô).

### 3.2. GriTS (Grid Table Similarity)
- Biến thể mới hơn, đánh giá theo dạng lưới (grid) thay vì cây, thường tính nhanh hơn TEDS.

### 3.3. Cell-level Precision/Recall/F1
- Coi mỗi ô là 1 đơn vị, so khớp (matching) ô dự đoán với ô ground truth theo vị trí, tính precision/recall trên số ô match đúng.
- Đơn giản, dễ hiểu hơn TEDS nhưng ít phản ánh lỗi cấu trúc tổng thể (như merge cell sai).

### 3.4. Đánh giá tách 2 tầng (rất hay gặp)
- **Structure-only**: chỉ chấm đúng/sai về số hàng, số cột, vị trí merge — không quan tâm text.
- **Structure + Content**: chấm cả cấu trúc lẫn nội dung từng ô đúng hay không.

### 3.5. Numeric cell accuracy
- Riêng với bảng số liệu tài chính (giống case của dự án này), hay tách thêm metric này vì 1 con số sai (ví dụ lệch 1 chữ số) trong bảng tài chính nghiêm trọng hơn nhiều so với 1 ô text thường bị sai chính tả nhẹ.

---

## 4. Nhận xét áp dụng cho dự án

Cách đánh giá dùng trong dự án (extract theo field cụ thể như tên, học vị + evidence/confidence) thực chất **né được việc phải đánh giá cấu trúc bảng thô bằng TEDS/GriTS** — vì chỉ cần đúng giá trị field cần lấy, không cần OCR tái dựng đúng 100% bảng gốc. Đây là lý do **task-based evaluation nhẹ và thực tế hơn** khi mục tiêu cuối là lấy thông tin cụ thể chứ không phải số hóa toàn bộ bảng.
