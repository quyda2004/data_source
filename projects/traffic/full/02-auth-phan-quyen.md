# Traffic — Xác thực & Phân quyền

## Xác thực & phân quyền — JWT HS256 + role_id

**Source:** https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_auth.py · https://github.com/quyda2004/check/blob/main/backend/app/core/config.py · https://github.com/quyda2004/check/blob/main/backend/app/models/user.py

**Cấp token:** login OAuth2 password flow (`POST /api/v1/auth/login`) → backend băm bcrypt để verify → ký JWT thuật toán **HS256** với `JWT_SECRET` + `ACCESS_TOKEN_EXPIRE_DAYS` từ config. Payload chứa `sub = user_id` và `role_id`.

**Xác thực HTTP:** dependency `get_current_user` (đọc header Bearer → decode → truy Mongo lấy `User`). Không có token / token sai → **401**.

**Xác thực WebSocket:** WebSocket không có header Authorization mặc định, nên hệ thống cho phép **truyền token qua query param `?token=...`**. Dependency `get_current_user_ws` xử lý riêng nhánh này cho endpoint `/ws/chat` và `/admin/ws/resources`.

**Phân quyền:** đơn giản bằng `role_id` (int):
- `role_id = 0` → **admin** (được vào `/admin/*`).
- Còn lại → user thường.

> **Hai endpoint chat song song có chủ đích:**
> - `POST /api/v1/chat` — yêu cầu auth + kiểm quota token LLM của user.
> - `POST /api/v1/chat_no_auth` — public, dùng `user_id = 9999` mặc định. Mục đích:
>   phục vụ **Telegram bot** (chưa có JWT cho từng người dùng Telegram) và cho demo
>   trên máy chiếu không cần login. Rõ ràng đây là **cửa sau demo**, không nên bật ở
>   production.

**Nhánh WebSocket frame hiện tại (`/ws/frames/{road_name}`, `/ws/info/{road_name}`):** để mở, không auth — vì frontend còn cần phát cho khách xem dashboard trực tiếp trong demo. Có comment sẵn trong `api_vehicles_frames.py` để bật auth khi triển khai thật.
