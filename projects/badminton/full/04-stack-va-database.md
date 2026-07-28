# Badminton — Ghi PostgreSQL, Stack & Test

## 1. Ghi dữ liệu xuống PostgreSQL — chỗ INSERT thật sự

**Source:** [backend/api/bookings.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/bookings.py) · [backend/api/deps.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/api/deps.py) · [backend/config.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/config.py)

Phân biệt rõ: **ai-service không ghi DB**, chỉ gọi HTTP. Chỗ ghi thật ở backend, cụ
thể ở `db.commit()`.

**Kết nối:** `backend/config.py` ghép `DATABASE_URL` dạng `postgresql+psycopg2://...`.
`backend/api/deps.py` tạo engine + session, inject vào request qua `Depends(get_db)`.

**Chỗ INSERT (trong `create_booking`)** — chuỗi 3 bước:
- `db.add(booking)` — đưa object vào session SQLAlchemy, **chưa** chạm DB (mới ở bộ nhớ).
- `db.commit()` — **đây mới là lúc dịch thành `INSERT INTO bookings ...` và ghi xuống
  PostgreSQL.**
- `db.refresh(booking)` — đọc lại bản ghi để lấy field DB tự sinh (`created_at` từ
  `server_default=func.now()`).

> **Phân biệt các luồng ghi khác:** hủy sân và dời lịch **chỉ sửa bản ghi có sẵn** nên
> **không có `db.add()`** — chỉ đổi thuộc tính (ví dụ `booking.status = "cancelled"`)
> rồi `db.commit()`. Chi tiết nhỏ hay bị hỏi: `add` chỉ cần cho bản ghi MỚI; sửa bản
> ghi cũ thì SQLAlchemy tự theo dõi thay đổi và `commit` là đủ.

**Tính giá:** `total_price = price_per_hour × số_giờ`, số giờ lấy từ
`(end_time - start_time).total_seconds() / 3600`. Reschedule tính lại theo khung giờ mới.

---

## 2. Stack công nghệ

**Source:** [backend/requirements.txt](https://github.com/quyda2004/AI_badminton/blob/main/backend/requirements.txt) · [ai_service/requirements.txt](https://github.com/quyda2004/AI_badminton/blob/main/ai_service/requirements.txt) · [requirements-dev.txt](https://github.com/quyda2004/AI_badminton/blob/main/requirements-dev.txt) · [docker-compose.yml](https://github.com/quyda2004/AI_badminton/blob/main/docker-compose.yml)

| Layer | Công nghệ |
|---|---|
| API Gateway | nginx:alpine (strip `/api`, proxy `/ai`, serve frontend) |
| Backend | FastAPI 0.136 + Uvicorn |
| ORM | SQLAlchemy 2.0 (Mapped / mapped_column) + psycopg2 |
| AI Service | FastAPI + google-genai 1.74 (Gemini) |
| Function calling | Native Gemini tools (6 tool) |
| MCP | mcp 1.27 (FastMCP) — wrap lại bộ tool |
| HTTP client (AI→backend) | httpx 0.28 |
| Auth | python-jose (JWT HS256, 60') + bcrypt |
| Validation | Pydantic 2.13 (+ email-validator) |
| DB quan hệ | PostgreSQL 16 |
| DB phi cấu trúc | MongoDB 7 (pymongo 4.17) |
| Frontend | Vanilla HTML/CSS/JS (SPA) — *ngoài phạm vi tổng hợp này* |
| Container | Docker + Docker Compose |
| Testing | pytest 9 |

**Biến môi trường chính** (`.env`): `POSTGRES_*`, `SECRET_KEY` (chung 2 service),
`GEMINIUS_API_KEY`, `GEMINIUS_MODEL` (`gemini-2.5-flash`), `MONGODB_URI`, `MONGODB_DB`,
`BACKEND_SERVICE_URL` (URL nội bộ AI→backend).

---

## 3. Test & Seed data — trạng thái thật

**Source:** [tests](https://github.com/quyda2004/AI_badminton/tree/main/tests) · [backend/scripts/seed.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/scripts/seed.py) · [backend/scripts/reset_db.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/scripts/reset_db.py) · [backend/scripts/check.py](https://github.com/quyda2004/AI_badminton/blob/main/backend/scripts/check.py)

**Có unit test** (pytest), thư mục `tests/` với `conftest.py` reset + seed DB sạch
trước khi chạy:

- `test_auth.py` — health, register (thành công / trùng email), login (đúng / sai pass
  / sai email), `/auth/me` (có / không token), kiểm tra response **không lộ**
  `hashed_password`.
- `test_courts.py` — list (chỉ active), tạo sân (admin OK / user 403 / no-token 401),
  availability (đúng 16 slot, court không tồn tại → 404).
- `test_bookings.py` — list, tạo (thành công / trùng slot → 409), get, cancel (thành
  công / hủy lần 2 → 400), reschedule.

**Seed** (`seed.py`): 10 user (9 user + 1 admin), 10 sân (standard/vip/premium, 1 sân
inactive), 10 booking (1 cancelled), 10 payment (có pending). Tài khoản test:
`nguyen.vana@gmail.com / 123` (user), `admin@badminton.com / admin` (admin).

`check.py` là script chẩn đoán: kiểm tra config, kết nối Postgres, tables, đếm dữ liệu,
verify bcrypt hash, cảnh báo nếu `SECRET_KEY` còn default.

> **Nói thẳng:** phần test là **logic xác định** (auth, phân quyền, luật trùng lịch,
> tính giá) — có input/output rõ ràng. **Chưa có đánh giá chất lượng đầu ra của LLM**
> (Gemini reply). Hướng mở rộng: human eval hoặc LLM-as-a-judge trên bộ test case chuẩn.
