# Traffic — RoadSetup Runtime + MongoDB Write + API Endpoints

## RoadSetup runtime — chỉnh live không cần restart

**Source:** [backend/app/api/v1/api_road_setups.py](https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_road_setups.py) · [backend/app/services/road_services/road_setup_runtime.py](https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/road_setup_runtime.py)

Đây là feature "vận hành viên chỉnh vùng ROI + vạch dừng ngay trên UI" — quan trọng vì mỗi camera lắp mỗi góc, không thể hard-code.

**Flow chỉnh setup:**
1. User mở `RoadSetupDialog` trong frontend → vẽ region ROI + blue_line + hộp đèn.
2. Frontend PUT `/api/v1/road-setups/{road_name}` với JSON zone/box theo resolution gốc.
3. Backend lưu vào Mongo collection `road_setups`.
4. User bấm "Reload" → POST `/api/v1/road-setups/{road_name}/reload`.
5. Backend gọi `road_setup_runtime.road_setup_to_runtime()` → scale zone/box về **600×400** → set flag trong `control_dict[road]` để process con **hot-reload config** ở đầu vòng loop tiếp theo.
6. Analyzer tiếp tục chạy, không cần restart backend.

> **Chi tiết `scale zone/box`** (`road_setup_runtime.py`, dòng 99–128): user vẽ zone
> trên preview frame ở resolution nào cũng được (ví dụ 1920×1080). Backend convert
> mọi toạ độ về hệ **600×400** cố định để analyzer chỉ phải xử lý ở 1 resolution.
> Nhờ vậy đo tốc độ (`meter_per_pixel`) không phải tính lại theo từng camera.

> **Lỗ hổng cần biết:** endpoint PUT road-setup hiện **để public** với comment
> "access control in code" — thực tế logic phân quyền chưa hoàn thiện. Ai biết URL
> đều PUT được → có thể ghi đè zone của tuyến khác. Cần enforce `role_id == 0` hoặc
> ownership check.

---

## Ghi dữ liệu xuống MongoDB — chỗ INSERT thật sự

**Source:** [backend/app/db/base.py](https://github.com/quyda2004/check/blob/main/backend/app/db/base.py) · [backend/app/models/](https://github.com/quyda2004/check/blob/main/backend/app/models/) · [backend/app/services/road_services/violation_sync.py](https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/violation_sync.py) · [backend/alembic/](https://github.com/quyda2004/check/blob/main/backend/alembic/)

Toàn bộ dữ liệu chính chạy trên **MongoDB (Motor + Beanie 2.1)**. Beanie là ODM async — model kế thừa `Document`, thao tác `.insert()` / `.save()` / `.find_one()` giống mongoose.

**Các collection chính:**

| Collection | Fields chính | Ghi bởi |
|---|---|---|
| `users` | username (unique), password (bcrypt), email (unique), phone_number (unique), role_id | `api_auth.py` khi register |
| `token_llm` | user_id (PK), token (int, default 5000) | `token_quota.py` khi trừ/khởi tạo |
| `chat_messages` | user_id, message, is_user, images[], extra_data, created_at | `api_chatbot.py` và `chat_history.py` |
| `violations` | road_name, vehicle_id, **event_id (unique)**, violation_type, plate_text, vehicle_image_b64, proof_b64, final_status, ocr_status, light_state_at_cross, turn_direction | `violation_sync.py` (upsert từ analyzer) |
| `road_setups` | road_name (unique), video_source, enabled, metric{}, violation{}, preview_frame_b64, updated_by, timestamps | `api_road_setups.py` khi user chỉnh trong UI |
| `traffic_stats` | road_name, count_car, count_motor, speed_car, speed_motor, congestion_level, time_frame, hour_start, hour_end | `seed.py` — dùng cho biểu đồ mật độ theo giờ trong tuần |

**Chỗ INSERT vi phạm (`violation_sync.py`, dòng 15–69):**
- **Không** làm `bulk_write` — vòng loop `for each violation → find_one → insert or save`. Lý do: batch nhỏ (thường <20 event / 3s), dễ debug, upsert key = `event_id` đảm bảo idempotent.
- **Không** chặn nếu 1 event lỗi — try/except từng cái, log rồi tiếp.

> **Chi tiết Alembic — hay bị hỏi:** `backend/alembic/` **có** 2 migration
> (`initial_migration_user_and_tokenllm`, `chat_messages_001_create_table`) nhưng đây
> là **DI SẢN từ nhánh SQL cũ**. Hiện app chạy 100% MongoDB, `DATABASE_URL` (Postgres)
> trong config vẫn giữ dạng optional để lát nữa nếu chuyển được thì đỡ đau. Nói
> thẳng: **Alembic không được dùng ở runtime** — chạy `alembic upgrade head` không
> ảnh hưởng gì tới demo. Nên xoá hoặc cách ly rõ ràng khi ship.

---

## API Endpoints đầy đủ

**Source:** [backend/app/main.py](https://github.com/quyda2004/check/blob/main/backend/app/main.py) · [backend/app/api/v1/](https://github.com/quyda2004/check/blob/main/backend/app/api/v1/)

**Backend (port 8000), tất cả prefix `/api/v1`:**

| Nhóm | Method | Path | Chức năng | Quyền |
|---|---|---|---|---|
| auth | POST | `/auth/register` | Đăng ký user | public |
| auth | POST | `/auth/login` | Login OAuth2 → trả `access_token` | public |
| auth | GET | `/auth/me` | Info user hiện tại | cần token |
| users | PUT | `/users/password` | Đổi mật khẩu | cần token |
| users | PUT | `/users/profile` | Cập nhật profile | cần token |
| chat | POST | `/chat` | Chat với AI (có quota) | cần token |
| chat | POST | `/chat_no_auth` | Chat demo/Telegram (user_id=9999) | public |
| chat | WS | `/ws/chat` | Chat streaming realtime | cần token (query) |
| chat-history | POST | `/chat/messages` | Lưu tin nhắn | cần token |
| chat-history | GET | `/chat/messages` | Lấy lịch sử (pagination) | cần token |
| chat-history | DELETE | `/chat/messages` | Xóa toàn bộ | cần token |
| chat-history | DELETE | `/chat/messages/{id}` | Xóa 1 tin | cần token |
| chat-history | GET | `/chat/messages/count` | Đếm tin nhắn | cần token |
| traffic | GET | `/roads_name` | List tuyến đang giám sát | public |
| traffic | GET | `/info/{road_name}` | Snapshot mật độ + tốc độ | public |
| traffic | WS | `/ws/frames/{road_name}` | Stream JPEG frame | public (comment sẵn để bật auth) |
| traffic | WS | `/ws/info/{road_name}` | Stream metric | public |
| traffic | GET | `/frames/{road_name}` | Ảnh JPEG (Authorization) | cần token |
| traffic | GET | `/frames_no_auth/{road_name}` | Ảnh JPEG demo | public |
| violations | GET | `/violations` | List vi phạm (filter road/type/from/to) | public |
| violations | GET | `/violations/count` | Đếm | public |
| violations | GET | `/violations/proof/event/{event_id}` | Ảnh bối cảnh theo event_id | public |
| violations | GET | `/violations/vehicle/event/{event_id}` | Ảnh xe theo event_id | public |
| violations | GET | `/violations/proof/{road_name}/{vehicle_id}` | Ảnh bối cảnh (legacy) | public |
| violations | GET | `/violations/vehicle/{road_name}/{vehicle_id}` | Ảnh xe (legacy) | public |
| violations | GET | `/violations/roads` | List tuyến có vi phạm | public |
| road-setup | GET | `/road-setups` | List setup | public |
| road-setup | GET | `/road-setups/{road_name}` | Chi tiết setup | public |
| road-setup | PUT | `/road-setups/{road_name}` | Cập nhật setup | public (access control trong code) |
| road-setup | POST | `/road-setups/{road_name}/reload` | Reload analyzer tuyến đó | public |
| road-setup | GET | `/roads/status` | Trạng thái tất cả tuyến | public |
| admin | GET | `/admin/resources` | Snapshot CPU/RAM/GPU/disk | **admin (role_id=0)** |
| admin | WS | `/admin/ws/resources` | Stream metrics hệ thống | **admin (role_id=0)** |

> **Chi tiết nên biết — vì sao nhiều endpoint `public`:** trong đồ án, phần lớn dữ
> liệu giao thông + vi phạm phải xem được **không cần đăng nhập** (demo cho hội đồng
> + chấm điểm không muốn login). Auth chỉ enforce trên chat (vì tốn tiền API LLM) và
> chỉnh sửa dữ liệu (users, chat history, admin). Trong triển khai thật ở đô thị nên
> đóng lại tất cả (`/road-setups PUT` đặc biệt phải admin), có comment sẵn ở
> `api_vehicles_frames.py` để bật auth cho WS frame.
