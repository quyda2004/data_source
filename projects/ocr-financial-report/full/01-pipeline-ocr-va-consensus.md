# OCR 9000 — Pipeline OCR & Consensus 3 LLM

## 1. Bối cảnh & vấn đề

- Có 9000 file báo cáo tài chính, định dạng không đồng bộ: một số file chỉ có text, một số chỉ là ảnh scan.
- Quyết định OCR toàn bộ để chuẩn hóa dữ liệu đầu vào.
- **Nguồn dữ liệu:** bộ 9000 file này đã có sẵn tại công ty từ trước khi tham gia dự án (dữ liệu nội bộ công ty thu thập, không phải tự crawl/thu thập).

---

## 2. Tiền xử lý dữ liệu

- OCR không xử lý trực tiếp trên file PDF → chuyển toàn bộ PDF sang ảnh trước.
- Vấn đề phát sinh: nhiều ảnh bị xoay ngang hoặc ngược chiều, gây lỗi khi OCR.

---

## 3. Xử lý hướng ảnh bằng ResNet-18

- Xây dựng mạng ResNet-18 (transfer learning) để phân loại 5 label hướng ảnh:
  - Label 0: 1 ảnh chứa 2 trang không cùng chiều
  - Label 1: ảnh đứng (chuẩn)
  - Label 2, 3, 4: các chiều còn lại (xoay/ngược)
- Từ 500.000 ảnh, sau khi qua mạng phát hiện và sửa thêm hơn 20.000 ảnh bị lỗi hướng.
- Mục tiêu: giảm thiểu sai số OCR do ảnh sai hướng gây ra.

### Vì sao chọn ResNet-18 cho bước sửa hướng ảnh?
- Bài toán chỉ cần phân loại hướng ảnh (5 label) — một bài toán classification tương đối đơn giản, không cần kiến trúc quá phức tạp/nặng.
- Dùng transfer learning trên ResNet-18 giúp tận dụng pretrained weight, giảm thời gian và dữ liệu cần train mà vẫn đảm bảo độ chính xác đủ tốt để lọc lỗi hướng trên tập ảnh lớn (500k ảnh).
- Mục tiêu là giảm thiểu sai số OCR gây ra bởi ảnh sai hướng, nên chỉ cần một classifier nhẹ, chạy nhanh trên số lượng ảnh lớn là phù hợp nhất, không cần overkill.

---

## 4. OCR — DeepSeek OCR

- Sử dụng **DeepSeek OCR** — SOTA tại thời điểm thực hiện, xử lý tốt các tác vụ OCR lúc đó (đây là lý do chính để chọn model này thay vì các OCR engine khác cùng thời điểm).

### Vì sao chọn DeepSeek OCR?
- Tại thời điểm thực hiện dự án, DeepSeek OCR là SOTA (state-of-the-art), xử lý tốt các tác vụ OCR văn bản tiếng Việt/báo cáo tài chính lúc đó.

---

## 5. Bài toán đánh giá kết quả (không thể label thủ công 9000 file)

**Vấn đề:** không thể đánh giá thủ công toàn bộ 9000 file vì khối lượng chữ quá lớn.

**Giải pháp:** sinh 2 bộ dữ liệu con từ bộ dữ liệu cha — nếu 2 bộ con đúng và có giá trị thì suy ra bộ cha có giá trị.

**Logic:** 2 bộ con này dùng chung toàn bộ pipeline (OCR + extract information) với 9000 file gốc. Nếu pipeline cho ra kết quả đúng/tin cậy trên 2 bộ con (đại diện cho toàn bộ tập), thì có cơ sở để tin rằng pipeline cũng hoạt động đúng trên toàn bộ 9000 file, thay vì phải đánh giá thủ công từng file một.

### Quy trình tạo bộ dữ liệu con
1. Xác định điểm chung của 9000 file: báo cáo thường niên công ty luôn có thông tin về giám đốc/người điều hành (tên, tháng năm sinh, học vị cao nhất) và hội đồng quản trị.
2. Lọc theo keyword: "giám đốc", "CEO",... lấy 150 từ trước và sau mỗi keyword.
3. Merge các đoạn trùng nhau: giữ phần trùng, gộp thêm phần không trùng.
4. Viết prompt engineering để extract information, có thêm 2 cột:
   - **Evidence**: trích đoạn text gốc làm bằng chứng cho kết quả extract, giúp truy vết lại nguồn khi cần kiểm tra.
   - **Confidence**: độ tự tin của model với kết quả extract, giúp nhận biết nhanh những trường hợp model không chắc chắn (dễ sai) để ưu tiên kiểm tra lại.
5. Chạy cùng prompt + text trên 3 LLM khác nhau: **GPT-4o**, **Claude Haiku 4.5**, **Claude 3.5**.
6. Tiêu chí đồng thuận: cả 3 model phải cho ra kết quả **trùng nhau tuyệt đối** mới được nhận. Các mẫu không trùng nhau giữa 3 model sẽ được đưa qua cho **human gán nhãn** thủ công.

---

## 6. Kết quả

| Bộ dữ liệu | Tỷ lệ mẫu đồng thuận (cả 3 LLM trùng nhau) |
|---|---|
| Giám đốc / người điều hành | 97.3% |
| Hội đồng quản trị | 98.6% |

- Đây là tỷ lệ mẫu mà cả 3 LLM cho kết quả trùng khớp nhau; phần còn lại (mẫu không trùng) được human gán nhãn riêng.
- Phần sai còn lại chủ yếu do lỗi OCR không extract tốt được nội dung gốc, không phải do lỗi ở bước prompt/LLM.
