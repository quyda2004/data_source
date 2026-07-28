# Badminton — Bối cảnh, Kiến trúc & Overlap Detection

**Repo:** https://github.com/quyda2004/AI_badminton

> **Ghi chú:** Các đường dẫn `Source` trỏ tới file thật trong repo theo dạng
> `.../blob/main/<đường-dẫn-file>`. File này cố ý **không chèn code** để nhẹ khi
> dùng làm data RAG — chi tiết triển khai xem trực tiếp ở file nguồn qua link.

---

## 1. Vì sao mình làm dự án này (bối cảnh)

**Source:** [README.md](https://github.com/quyda2004/AI_badminton/blob/main/README.md)

Ý tưởng xuất phát từ nhu cầu đời thường: đặt sân cầu lông qua form (chọn sân, ngày, giờ, bấm nút) tuy chạy được nhưng khô khan và nhiều bước. Mình muốn thử hướng khác — **đặt sân bằng hội thoại tiếng Việt tự nhiên**, kiểu nói chuyện với lễ tân: *"Đặt sân 1 ngày mai lúc 8h"*, *"Hủy sân 2 chiều nay giúp mình"* — hệ thống tự hiểu, tự tra, tự thực hiện.

Điểm muốn chứng minh không phải "làm ra AI thông minh hơn" (bộ não vẫn là Gemini của Google), mà là **biến chatbot chung chung thành AI agent chuyên biệt hành động thật trong hệ thống nghiệp vụ**: truy vấn dữ liệu thật, đặt/hủy sân thật, phân quyền theo từng user, và giấu chi tiết kỹ thuật (ID nội bộ) để hội thoại mượt như nói với người thật.

Về kỹ thuật, xây theo **kiến trúc microservice**: phần nghiệp vụ (REST API) và phần AI (chat) là hai service độc lập, giao tiếp qua HTTP. Đây là quyết định thiết kế trung tâm, mọi thứ khác xoay quanh nó.

---

## 2. Kiến trúc tổng thể — 5 khối

**Source:** [docker-compose.yml](https://github.com/quyda2004/AI_badminton/blob/main/docker-compose.yml) · [nginx/nginx.conf](https://github.com/quyda2004/AI_badminton/blob/main/nginx/nginx.conf) · [architecture.tex](https://github.com/quyda2004/AI_badminton/blob/main/architecture.tex) · [README.md](https://github.com/quyda2004/AI_badminton/blob/main/README.md)

Hệ thống gồm 5 khối chạy trong Docker Compose. Mọi request từ frontend đi qua **nginx làm API Gateway** trước, rồi nginx chia 2 nhánh.

| Service | Port | Công nghệ | Trách nhiệm |
|---|---|---|---|
| `nginx` | 80 | nginx:alpine | API Gateway + serve frontend static |
| `backend` | 8000 | FastAPI + SQLAlchemy | Auth, Courts, Bookings, Payments, Admin |
| `ai` | 8001 | FastAPI + Gemini API | AI Chat, Function Calling, MCP Server |
| `postgres` | 5432 | PostgreSQL 16 | Dữ liệu nghiệp vụ (users, courts, bookings, payments) |
| `mongodb` | 27017 | MongoDB 7 | Lịch sử chat (chat history) |

**Cách nginx định tuyến:**
- `/api/*` → **strip prefix `/api`** → đẩy sang `backend:8000`. Frontend gọi `/api/courts` thì backend nhận `/courts`.
- `/ai/*` → **giữ nguyên path** → đẩy sang `ai:8001`. Frontend gọi `/ai/chat` thì ai-service nhận đúng `/ai/chat`.
- `/health` → nginx tự trả JSON của gateway.
- `/` → serve frontend static (SPA vanilla JS).

> **Nguyên tắc thiết kế cốt lõi (điểm ăn điểm nhất khi phỏng vấn):** ai-service
> **KHÔNG truy cập thẳng vào PostgreSQL** của backend. Khi AI cần đặt sân, nó **gọi
> ngược lại REST API của backend qua HTTP**, y hệt một client bình thường. Nhờ vậy
> toàn bộ logic nghiệp vụ (check trùng giờ, tính giá, phân quyền) chỉ tồn tại **một
> chỗ duy nhất** ở backend, không lặp. Sửa một luật chỉ sửa một nơi.

**Hai service tin nhau bằng cách nào:** cả `backend` và `ai` cùng đọc chung biến
`SECRET_KEY`. Nhờ chung khóa bí mật này, ai-service **tự giải mã JWT do backend cấp**
mà không cần gọi lại backend để hỏi "token này của ai".

---

## 3. Cơ chế check trùng lịch đặt sân (overlap detection)

**Source:** [backend/api/bookings.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/bookings.py) · [backend/api/courts.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/courts.py)

Luật nghiệp vụ quan trọng nhất, dùng **chung một công thức** ở 3 chỗ.

**Công thức overlap:** Hai khoảng thời gian đè lên nhau **khi và chỉ khi**
`start_cũ < end_mới` **VÀ** `end_cũ > start_mới`. Ví dụ booking cũ 8h–10h, đặt mới
9h–11h → trùng (8<11 và 10>9). Đặt mới 10h–12h → không trùng (10>10 sai, khít mép OK).
Mọi booking `cancelled` bị loại khỏi phép kiểm.

**Chỗ 1 — tạo booking** (`create_booking`): tìm thấy booking chồng giờ → trả **409
Conflict**. Có test `test_create_booking_duplicate_slot`.

**Chỗ 2 — dời lịch** (`reschedule_booking`): giống hệt nhưng thêm điều kiện **loại
chính nó** (`Booking.id != booking.id`) để không tự trùng với bản thân. Thiếu điều
kiện này thì reschedule luôn báo trùng.

**Chỗ 3 — xem lịch trống** (`get_availability` trong courts.py): không chặn đặt sân
mà để hiển thị. Lặp từng giờ 6h→22h (16 slot), mỗi giờ dùng công thức overlap gán
`available = True/False`.

> **Lỗ hổng cần biết (phỏng vấn hay xoáy) — race condition:** cách làm hiện tại là
> **check-rồi-mới-ghi**, không có khóa. Nếu 2 request đặt cùng slot chạy đồng thời,
> cả hai đều thấy "trống" trước khi ai kịp ghi → lọt cả hai (double-booking). Trong
> demo gần như không xảy ra. Khắc phục production: (a) thêm **exclusion constraint**
> ở PostgreSQL (range type + `EXCLUDE`), hoặc (b) dùng **`SELECT ... FOR UPDATE`**
> khóa dòng trong transaction. Biết được giới hạn này là điểm cộng.

---

## 4. Hai database — vì sao tách PostgreSQL và MongoDB

**Source:** [backend/models](https://github.com/quyda2004/AI_badminton/tree/main/backend/models) · [ai_service/database.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/database.py) · [docker-compose.yml](https://github.com/quyda2004/AI_badminton/blob/main/docker-compose.yml)

Quyết định "dùng đúng DB cho đúng loại dữ liệu":

- **PostgreSQL** giữ dữ liệu nghiệp vụ (`users`, `courts`, `bookings`, `payments`):
  có cấu trúc, cần **ràng buộc khóa ngoại và tính đúng đắn** (booking tham chiếu user
  + court; payment tham chiếu booking). Tiền dùng `Numeric(10, 2)`.
- **MongoDB** giữ **lịch sử chat** (`chat_history`): phi cấu trúc, ghi nhiều, không
  cần join phức tạp — hợp NoSQL. Mỗi document: `user_id`, `session_id`, `role`,
  `content`, `created_at`.

Các model quan hệ chính:
- `User`: id, email (unique), phone, full_name, hashed_password, role (`user`/`admin`), timestamps.
- `Court`: id, name, type (`standard`/`vip`/`premium`), price_per_hour, status (`active`/`inactive`), description.
- `Booking`: id, user_id (FK), court_id (FK), start_time, end_time, status (`confirmed`/`cancelled`), total_price, timestamps.
- `Payment`: id, booking_id (FK), amount, method (`cash`/`banking`/`momo`), status (`paid`/`pending`), paid_at, created_at.

---

## 5. Điểm khác biệt so với chatbot thị trường + Hướng phát triển

**Khác biệt thật sự (đã có):** bộ não vẫn là Gemini nên **không thể "thông minh hơn
Gemini"**. Khác biệt nằm ở chỗ biến chatbot chung chung thành **agent chuyên biệt hành
động được**: (1) grounding + thực thi có side-effect (đọc dữ liệu sân thật, đặt/hủy
sân thật trong Postgres — không chỉ "nói"); (2) danh tính & phân quyền (mỗi thao tác
gắn JWT của user, không cross-user); (3) giấu phức tạp nội bộ (không hỏi ID, tự tra và
khớp qua `cancel_booking_by_info`).

**Điểm yếu hiện tại (nên tự nói ra khi phỏng vấn):** chưa có memory dài hạn / cá nhân
hóa theo thói quen user; chưa xử lý tốt câu mơ hồ / gợi ý chủ động ("sân nào rẻ mà tối
nay còn trống?"); chưa giới hạn vòng lặp tool; chưa có guardrail chống prompt injection;
chưa có luồng tạo payment qua API (payment hiện chỉ được seed sẵn); chưa có tool
reschedule cho AI.

**Hướng phát triển gợi ý (xếp theo độ dễ + độ ăn điểm):**
1. **Gợi ý chủ động:** thêm tool "tìm slot rẻ nhất còn trống hôm nay" — tái dùng
   `get_courts` + `check_availability`. Biến bot từ người nhận lệnh thành người tư vấn.
2. **Cá nhân hóa bằng lịch sử:** đã có chat trong MongoDB, đọc thêm lịch sử booking để
   gợi ý "đặt lại sân quen như tuần trước không?".
3. **Nhắc lịch & nhắc thanh toán chủ động:** nối với lỗ hổng payment — bot nhắc
   "booking mai chưa thanh toán".
4. **Guardrails + giới hạn vòng lặp + xử lý mơ hồ:** ít hào nhoáng nhưng cho thấy tư
   duy production, độ tin cậy.

> **Lời khuyên phỏng vấn:** chọn 1–2 hướng làm cho tới thay vì rải hết. Gợi ý mạnh
> nhất: *gợi ý chủ động* (dễ demo, ấn tượng ngay) + *guardrails* (tư duy kỹ sư).
