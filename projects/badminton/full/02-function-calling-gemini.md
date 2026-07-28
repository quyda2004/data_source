# Badminton — JWT Liên Service, Flow Đặt Sân AI, Function Calling & API Endpoints

## 1. Xác thực & phân quyền — JWT liên service

**Source:** [backend/security.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/security.py) · [backend/api/deps.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/deps.py) · [backend/api/auth.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/auth.py) · [ai_service/router.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/router.py) · [backend/config.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/config.py) · [ai_service/config.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/config.py)

**Cấp token:** `backend/security.py` băm mật khẩu bằng bcrypt và ký JWT thuật toán
**HS256, hết hạn 60 phút**. Khi login thành công (`backend/api/auth.py`), backend nhét
`sub = user.id` và `role` vào token rồi trả về `access_token`.

**Xác thực ở backend** (`backend/api/deps.py`) — chuỗi 2 dependency xếp tầng:
- `get_current_user`: đọc header Bearer → decode token → lấy `sub` → query DB ra
  `User`. Không có token / token sai / user không tồn tại → **401**.
- `get_admin_user`: gọi `get_current_user` trước, rồi kiểm tra `role == "admin"`,
  nếu không → **403**. Dùng cho `POST /courts` và toàn bộ router `/admin`.

**Xác thực ở ai-service** (`ai_service/router.py`) — hàm `_get_user` tự decode JWT
bằng chung `SECRET_KEY`, chỉ lấy `user_id`, và **giữ lại nguyên token** để lát nữa
forward xuống backend. Không gọi backend để xác thực.

> **Hệ quả bảo mật quan trọng:** khi ai-service gọi API backend, nó đính kèm **chính
> JWT của user đó**. Nên backend vẫn enforce quyền như thường — **user A không thể
> xem/hủy/dời booking của user B** kể cả khi thao tác đi qua AI. AI không có "siêu
> quyền"; nó chỉ hành động trong đúng phạm vi quyền của user đang đăng nhập.

---

## 2. Flow chính — Đặt sân bằng AI

**Source:** [ai_service/router.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/router.py) · [ai_service/gemini_client.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/gemini_client.py) · [ai_service/tools.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/tools.py) · [ai_service/database.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/database.py) · [backend/api/bookings.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/bookings.py)

Luồng "demo tủ". Kịch bản: user nhắn **"Đặt sân 1 ngày mai lúc 8h"**.

**Sơ đồ tuần tự:**
```
User → nginx → ai-service (/ai/chat) [kèm JWT]
   │
   ├─ [B2] decode JWT cục bộ → user_id (không gọi backend)
   ├─ [B3] load history (MongoDB) + prompt → gửi Gemini (kèm 6 tool)
   ├─ [B4] VÒNG LẶP: Gemini trả function_call → ai-service thực thi tool
   │        → gọi HTTP backend → nhét kết quả lại cho Gemini → lặp
   │        (get_courts → check_availability → create_booking)
   │        → Gemini trả text thường → dừng
   ├─ [B5] mỗi tool = 1 HTTP call xuống backend (kèm JWT) → backend chạm PostgreSQL
   └─ [B6] Gemini sinh reply tiếng Việt → lưu 2 message vào MongoDB → trả về user
```

**Bước 1 — vào endpoint `/ai/chat`** (`ai_service/router.py`): nhận message + JWT,
gọi `chat_with_user`.

**Bước 2 — decode JWT:** dependency `_get_user` chạy trước, decode token cục bộ lấy
`user_id`.

**Bước 3 — nạp ngữ cảnh & gọi Gemini** (`ai_service/gemini_client.py`): load lịch sử
phiên từ MongoDB, ghép prompt mới, gửi Gemini kèm `SYSTEM_PROMPT` và khai báo 6 tool.
`SYSTEM_PROMPT` là bộ luật ép Gemini đi đúng quy trình (get_courts → check_availability
→ báo giá → create_booking) và **không bao giờ hỏi user `court_id` / `booking_id`** —
thông tin nội bộ user không biết. Luật chốt: *"Không bao giờ tự đoán ID — phải lấy từ tools."*

**Bước 4 — vòng lặp function-calling (trái tim của luồng):** điểm cốt lõi là **Gemini
KHÔNG tự gọi HTTP**. Nó chỉ *quyết định* muốn gọi tool nào (trả về `function_call`).
ai-service mới là đứa thực thi (qua bộ dispatch `_run_tool`), rồi đưa kết quả *trở lại*
cho Gemini ở vòng lặp kế tiếp. Cứ lặp tới khi Gemini trả về câu chữ bình thường thì dừng.

**Bước 5 — tool gọi HTTP xuống backend** (`ai_service/tools.py`): mỗi tool là 1 cú
`httpx` tới đúng endpoint backend, **kèm JWT của user** trong header Authorization.
Ở backend, khi `create_booking` được gọi, chính luật check trùng lịch + tính giá chạy,
rồi ghi xuống PostgreSQL. *Chi tiết timezone:* tool dùng `VN_TZ` (+07:00), user nói
"8h" là giờ VN, tool ghép thành datetime có tzinfo rồi gửi ISO string xuống — không
lệch múi giờ.

**Bước 6 — Gemini sinh reply → lưu MongoDB → trả về:** vòng lặp thoát với reply tiếng
Việt, lưu cả tin user lẫn reply vào collection `chat_history` (MongoDB) rồi trả về
frontend.

> **Câu chốt để nhớ cả luồng:** *"Gemini là bộ não ra quyết định gọi tool nào, còn
> ai-service là bộ tay chân thực thi tool đó. Mọi thao tác dữ liệu đều đi qua backend
> chứ AI không bao giờ chạm thẳng vào database."*

---

## 3. Bộ Tools & Function Calling (native Gemini)

**Source:** [ai_service/gemini_client.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/gemini_client.py) · [ai_service/tools.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/tools.py)

Dùng **native function calling của Gemini**: 6 tool khai báo schema đầy đủ, Gemini
**tự quyết định** gọi tool nào trong vòng lặp.

| Tool | Tham số | Chức năng | Endpoint backend gọi |
|---|---|---|---|
| `get_courts` | — | Lấy danh sách sân active | `GET /courts` |
| `check_availability` | `court_id`, `date_str` | Slot trống + giá theo ngày | `GET /courts/{id}/availability` |
| `create_booking` | `court_id`, `date_str`, `start_hour`, `end_hour` | Đặt sân | `POST /bookings` |
| `get_my_bookings` | — | List booking của user | `GET /bookings` |
| `cancel_booking` | `booking_id` | Hủy theo ID | `PUT /bookings/{id}/cancel` |
| `cancel_booking_by_info` | `court_name`, `date_str`, `start_hour` | Hủy theo tên sân + ngày + giờ | `GET /bookings` rồi `PUT .../cancel` |

> **Điểm thiết kế UX cho AI — `cancel_booking_by_info`:** user không bao giờ biết
> `booking_id`. Tool này cho hủy bằng ngôn ngữ tự nhiên: tự `GET /bookings`, lọc bỏ
> booking đã hủy, khớp **tên sân (chứa một phần cũng được)** + **ngày** + **giờ bắt
> đầu** (có chuyển tzinfo về giờ VN để so), tìm ra ID rồi mới hủy. Đây chính là cách
> "giấu chi tiết kỹ thuật" để hội thoại tự nhiên.

> **Nói thẳng cho đúng — giới hạn hiện tại:**
> - Vòng lặp function-calling là `while True` **chưa có giới hạn số lần lặp**. Nếu
>   Gemini kẹt gọi tool liên tục thì về lý thuyết có thể loop dài. Cần thêm guard.
> - **Chưa có tool `reschedule`** cho AI: backend có endpoint dời lịch nhưng Gemini
>   không gọi được (không nằm trong 6 tool). Qua chat chỉ đặt/hủy được.
> - **Chưa có tầng chống prompt injection** (JWT vẫn chặn được thao tác cross-user,
>   nhưng chưa có guardrail chủ động ở tầng prompt).

---

## 4. API Endpoints đầy đủ

**Source:** [backend/api/main.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/main.py) · [backend/api/auth.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/auth.py) · [backend/api/courts.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/courts.py) · [backend/api/bookings.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/bookings.py) · [backend/api/payments.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/payments.py) · [backend/api/admin.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/admin.py) · [ai_service/router.py](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/router.py)

**Backend (port 8000):**

| Nhóm | Method | Path | Chức năng | Quyền |
|---|---|---|---|---|
| auth | POST | `/auth/register` | Đăng ký user | public |
| auth | POST | `/auth/login` | Đăng nhập → trả `access_token` | public |
| auth | GET | `/auth/me` | Thông tin user hiện tại | cần token |
| courts | GET | `/courts` | List sân `status=active` | public |
| courts | POST | `/courts` | Tạo sân | **admin** |
| courts | GET | `/courts/{id}/availability?date=` | 16 slot giờ 6h→22h, trống/bận | public |
| bookings | GET | `/bookings` | List booking của chính user | cần token |
| bookings | POST | `/bookings` | Đặt sân (check trùng → 409) | cần token |
| bookings | GET | `/bookings/{id}` | Chi tiết 1 booking | cần token |
| bookings | PUT | `/bookings/{id}/cancel` | Hủy booking | cần token |
| bookings | PUT | `/bookings/{id}/reschedule` | Dời lịch (check trùng lại) | cần token |
| payments | GET | `/payments/{booking_id}` | List payment của 1 booking | cần token |
| admin | GET | `/admin/stats` | Thống kê (doanh thu, số booking, số user) | **admin** |
| admin | GET | `/admin/bookings` | Toàn bộ booking hệ thống | **admin** |
| health | GET | `/health` | `{"status":"ok"}` | public |

**AI Service (port 8001):**

| Method | Path | Chức năng |
|---|---|---|
| POST | `/ai/chat` | Gửi tin nhắn, nhận reply từ Gemini |
| GET | `/ai/history/{session_id}` | Lịch sử 1 phiên chat |
| GET | `/ai/history` | Toàn bộ lịch sử của user |
| GET | `/health` | Health check |

> **Chi tiết `admin/stats`:** `total_revenue` chỉ tính booking **không** bị hủy
> (`status != "cancelled"`), mốc "hôm nay" theo giờ VN.
>
> **Chi tiết `get_availability`:** hiện chỉ kiểm tra sân **tồn tại** (404 nếu không),
> **chưa** kiểm tra sân có `active` không — về lý thuyết vẫn xem được lịch của sân
> `inactive` (như `court_10` đang bảo trì). Điểm có thể siết lại.
