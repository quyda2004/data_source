# Traffic — Bối cảnh & Kiến trúc tổng thể

**Repo:** https://github.com/quyda2004/check

> **Ghi chú:** Các đường dẫn `Source` trỏ tới file thật trong repo theo dạng
> `.../blob/main/<đường-dẫn-file>`. Nếu nhánh mặc định là `master` hoặc `dev`, thay
> `/blob/main/` cho khớp. File này cố ý **không chèn code** để nhẹ khi dùng làm data
> RAG — chi tiết triển khai xem trực tiếp ở file nguồn qua link.

---

## 1. Vì sao mình làm dự án này (bối cảnh)

**Source:** [README.md](https://github.com/quyda2004/check/blob/main/README.md) · [bao_cao_do_an.md](https://github.com/quyda2004/check/blob/main/bao_cao_do_an.md)

Đây là **đồ án tốt nghiệp** (ngành Khoa học Dữ liệu, ĐH KHTN – ĐHQG TP.HCM). Ý tưởng xuất phát từ việc các demo Computer Vision giao thông trên mạng thường chỉ dừng ở một mảnh nhỏ: hoặc là đếm xe, hoặc là đo tốc độ, hoặc là detect biển số — chạy trong notebook, xem video một lần rồi thôi. Mình muốn ghép mọi thứ thành **một pipeline hoàn chỉnh chạy realtime trên web**: quay/mở nhiều video tuyến đường cùng lúc → phân tích liên tục → hiển thị lên dashboard → cho user hỏi bằng chatbot tiếng Việt như hỏi CSGT trực ban.

Điểm muốn chứng minh không phải "làm AI thông minh hơn YOLO" (backbone vẫn là YOLOv8 + OpenVINO, LLM vẫn là Gemini), mà là **ghép nhiều mô-đun thị giác + AI + web + realtime thành một hệ chạy được và hỏi được**: detect xe → tracking → đo tốc độ → phát hiện vi phạm đèn đỏ → OCR biển số → lưu bằng chứng → cho ReAct Agent tra cứu và trả lời bằng ngôn ngữ tự nhiên. Toàn bộ chạy được **9–10 FPS trên CPU phổ thông** và song song nhiều tuyến đường.

Về kỹ thuật, trung tâm là quyết định **mỗi tuyến đường chạy trên một process riêng**, chia sẻ dữ liệu qua shared memory của Python `multiprocessing.Manager`, còn FastAPI backend + WebSocket chỉ đóng vai trò "cửa sổ" đọc ra để đẩy về frontend. Quyết định này ảnh hưởng đến gần như mọi thứ khác (khởi tạo model 1 lần, cách analyzer expose data, cách chatbot đọc dữ liệu live thay vì query DB).

---

## 2. Kiến trúc tổng thể — 3 khối

**Source:** [backend/app/main.py](https://github.com/quyda2004/check/blob/main/backend/app/main.py) · [frontend/vite.config.ts](https://github.com/quyda2004/check/blob/main/frontend/vite.config.ts) · [backend/app/db/base.py](https://github.com/quyda2004/check/blob/main/backend/app/db/base.py) · [README.md](https://github.com/quyda2004/check/blob/main/README.md)

Hệ thống gồm 3 khối chính — **cố ý không dùng API Gateway riêng** để giữ setup gọn cho demo đồ án.

| Service | Port | Công nghệ | Trách nhiệm |
|---|---|---|---|
| `backend` | 8000 | FastAPI + Uvicorn + Beanie ODM | REST API + WebSocket, chạy multiprocessing analyzer, gọi Gemini/Roboflow |
| `frontend` | 5173 | React 19 + Vite 7 + TypeScript + shadcn/ui | Dashboard realtime, chat UI, trang vi phạm, trang admin |
| `mongodb` | 27017 | MongoDB (Motor + Beanie 2.1) | Users, ChatMessage, TokenLLM, Violation, RoadSetup, TrafficStat |

**Cách frontend gọi backend:**
- **HTTP REST** (`/api/v1/*`) cho auth, chat, history, violations, road-setups, admin stats.
- **WebSocket** (`/api/v1/ws/*`) cho realtime: `ws/frames/{road}` stream MJPEG-like, `ws/info/{road}` stream mật độ/tốc độ, `ws/chat` stream token LLM, `admin/ws/resources` stream CPU/RAM.
- Base URL config ở `frontend/src/config.ts`, dev mode có `MOCK_TOKEN` để đỡ phải login khi test UI.

> **Nguyên tắc thiết kế cốt lõi (điểm ăn điểm nhất khi phỏng vấn):** analyzer video
> **KHÔNG chạy trong request handler** của FastAPI. Nó khởi tạo **một lần** ở lifespan
> `startup` và chạy trong các process con độc lập. Endpoint chỉ đóng vai trò **đọc
> ra** từ shared dict/list (`state.analyzer.info_dict`, `state.analyzer.frame_dict`,
> `state.analyzer.violations_list`). Nhờ vậy, 1000 client cùng vào xem frame vẫn chỉ
> có **1 process xử lý video** — không nhân đôi tải AI theo số user.

**Auto-start khi backend lên** (`backend/app/main.py` lifespan `startup`, dòng 77–106):
1. Kết nối MongoDB qua Motor + init Beanie với 5 model.
2. Load `RoadSetup` từ DB, dựng runtime config cho từng tuyến.
3. Khởi tạo `AnalyzeOnRoadForMultiprocessing` → `run_multiprocessing()` fork process con mỗi tuyến.
4. Start background task `violation_sync_loop` đồng bộ vi phạm ra Mongo mỗi 3s.
