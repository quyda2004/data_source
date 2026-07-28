# Skills Detail — Bản đầy đủ có evidence link

> Bản này chi tiết hơn `skills-map.md` — mỗi skill kèm **link source thẳng vào
> code repo** để LLM/reviewer verify được. File này nạp cùng phần tĩnh trong
> system prompt (có prompt caching).

---

## Backend / API Development

### FastAPI + Uvicorn
- **traffic** (`backend/app/main.py`, `backend/app/api/v1/`) — https://github.com/quyda2004/check/blob/main/backend/app/main.py
- **badminton** (`backend/api/main.py`) — https://github.com/quyda2004/AI_badminton/blob/main/backend/api/main.py
- **ai-coworker** (`app/main.py`, `app/api/routes/`) — https://github.com/quyda2004/AI_coworker/blob/main/app/main.py

### WebSocket streaming (native FastAPI)
- **traffic**: 4 kênh — https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_vehicles_frames.py
  - `/ws/frames/{road}` MJPEG-like bytes
  - `/ws/info/{road}` metric
  - `/ws/chat` LLM streaming token
  - `/admin/ws/resources` CPU/RAM

### JWT HS256 authentication
- **traffic** (`backend/app/api/v1/api_auth.py`) — https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_auth.py
  - Payload: `sub = user_id`, `role_id`
  - WS truyền token qua query param `?token=...` (không có Authorization header)
- **badminton** (`backend/security.py`, `backend/api/deps.py`) — https://github.com/quyda2004/AI_badminton/blob/main/backend/security.py
  - Hết hạn 60 phút, `role` (user/admin)

### Multi-service JWT (chung SECRET_KEY)
- **badminton** — `backend` và `ai` cùng đọc `SECRET_KEY` → ai-service tự decode JWT lấy `user_id` mà không phải call backend hỏi. Khi ai-service gọi backend → attach nguyên JWT của user → backend enforce quyền như thường. https://github.com/quyda2004/AI_badminton/blob/main/ai_service/router.py

---

## AI / LLM Engineering

### LangGraph `create_react_agent` (single-agent + tool)
- **traffic** — 3 tool, `recursion_limit=6`, `pre_model_hook` cắt history 2000 token. https://github.com/quyda2004/check/blob/main/backend/app/services/chat_services/ChatBotAgent.py
- Danh sách tuyến thật **hard-inject vào prompt** để LLM không bịa tên.
- Backend tự regex bóc URL + tự đính frame → giảm 1 vòng LLM format URL.

### LangGraph multi-node graph (7 node)
- **ai-coworker** (`app/engine/graph.py`, `app/engine/supervisor.py`, `app/engine/director.py`) — https://github.com/quyda2004/AI_coworker/blob/main/app/engine/graph.py
  - Nodes: Supervisor, Director, CEO, CHRO, RegionalManager, Mentor, SafetyBlock.
  - Flow: `entry=Supervisor → Director → conditional edges → [agent] → END` (one-turn per invoke).
  - Supervisor `temperature=0.0`, `max_output_tokens=20`.

### Native Gemini function calling
- **badminton** (`ai_service/gemini_client.py`, `ai_service/tools.py`) — https://github.com/quyda2004/AI_badminton/blob/main/ai_service/tools.py
  - 6 tool khai báo schema đầy đủ, Gemini tự quyết định gọi.
  - Vòng lặp: LLM → `function_call?` → dispatch → nhét kết quả lại → lặp.
  - Điểm khéo: `cancel_booking_by_info(court_name, date, start_hour)` — giấu `booking_id`.

### Multi-LLM consensus evaluation
- **ocr-financial-report** — GPT-4o + Claude Haiku 4.5 + Claude 3.5 chạy cùng prompt, phải trùng tuyệt đối mới nhận. Kết quả: 97.3% (giám đốc), 98.6% (HĐQT). Phần không trùng → human label.

### MCP Server (FastMCP)
- **badminton** (`ai_service/mcp_server.py`) — https://github.com/quyda2004/AI_badminton/blob/main/ai_service/mcp_server.py
  - Wrap 6 tool → chạy `python -m ai_service.mcp_server`.
  - Trạng thái thật: token rỗng → tool cần auth sẽ fail (scaffolding, chưa production-ready).

### Prompt caching (`cache_control: ephemeral`)
- **portfolio** — nạp `index.json + meta-all.md + skill-detail.md` với `cache_control`, từ call 2 trở đi tính cache-read thay vì full input.

---

## RAG / Vector Database

### FAISS per-agent isolated index
- **ai-coworker** (`app/knowledge/retriever.py`, `app/db/vector/faiss_store.py`, `app/knowledge/ingest.py`) — https://github.com/quyda2004/AI_coworker/blob/main/app/knowledge/retriever.py
  - 4 index: `ceo/`, `chro/`, `regional/`, `shared/`.
  - Chunk 500 ký tự, overlap 80.
  - Metadata: `role_access`, `topic`.
  - Agent bị hỏi sai lĩnh vực → tự chuyển user sang agent đúng.

---

## Computer Vision

### YOLOv8 detection (train + deploy)
- **traffic** — 3 model train theo pipeline paper tham chiếu:
  - **Vehicle** (YOLOv8l 6-class, mAP@0.5 = 0.909 theo paper [1] Sadik et al.) — https://github.com/quyda2004/check/blob/main/refer_paper/paper1_vehicle_detection_VI.md
  - **Helmet** (YOLOv8 binary, mAP@0.5 = 91% theo paper [4] Aquitan et al.)
  - **Plate** (YOLOv8 1-class, F1 = 0.9968 theo paper [5] Moussaoui et al.)
  - Train Google Colab, Tesla T4 15GB.
  - Notebook convert model: https://github.com/quyda2004/check/blob/main/backend/app/ai_models/model%20N/cvt_model.ipynb

### ByteTrack multi-object tracking
- **traffic** — dùng qua Ultralytics `.track(..., tracker="bytetrack.yaml")`, `confidence=0.15` (thấp để giữ box confidence thấp cho ByteTrack "vớt" xe bị che). https://github.com/quyda2004/check/blob/main/refer_paper/bytetrack_tomtat.md

### Monocular 2D positioning (đo tốc độ)
- **traffic** — `meter_per_pixel` config trong `road_setup` (0.12 m/px mặc định), `region` polygon giới hạn vùng đo tốc độ. Dùng `solutions.SpeedEstimator` của Ultralytics. Paper [3] Lian et al.: Min accuracy 95.1%, Avg 97.6%. https://github.com/quyda2004/check/blob/main/refer_paper/monocular_speed_measurement_tomtat.md

### OpenVINO INT8 quantization
- **traffic** — path production: `ai_models/model N/openvino models/best_int8_openvino_model`. CPU-only 9-10 FPS. https://github.com/quyda2004/check/blob/main/backend/app/core/config.py

### EasyOCR + Gemini Vision fallback
- **traffic** (`backend/app/modules/plate_ocr_service.py` dòng 44-138) — https://github.com/quyda2004/check/blob/main/backend/app/modules/plate_ocr_service.py
  - EasyOCR fail → gọi Gemini với prompt trả JSON `{plate_text, vehicle_type}`.

### ResNet-18 transfer learning (5-label classification hướng ảnh)
- **ocr-financial-report** — sửa 20.000+ ảnh sai hướng trong 500.000 ảnh.

### DeepSeek OCR
- **ocr-financial-report** — chọn vì SOTA thời điểm dự án.

### Table evaluation metrics (kiến thức nền)
- **ocr-financial-report** — TEDS (Tree Edit Distance-based Similarity), GriTS (Grid), Cell-level Precision/Recall/F1, Numeric cell accuracy (cho tài chính).

---

## Signal Processing / Time-Series

### 1-D CNN QRS detection
- **af-detection-ecg** — Label ±40ms quanh R-peak, sample -100ms→+300ms. Accuracy 99.92% trên 22 người test.
- Reference: Šarlija et al. (ISPA 2017).

### Baseline wander removal
- **af-detection-ecg** — Downsample 360→50Hz linear estimation, khung 4s, QRS được coi là outlier nên không mất khi trừ baseline.

### K-means post-processing cho R-peak
- **af-detection-ecg** — Cửa sổ 80ms, >5 điểm label 1 → lấy tâm cụm.

### DeepCNN 24 lớp + BiLSTM local/global
- **af-detection-ecg** — 3 block × 8 conv (filter 32/64/128), 2 lớp BiLSTM (lớp 1 local, lớp 2 global), pooling hệ số 4. Accuracy 98.3%, đạt AAMI EC57.
- Reference: Cheng et al. (2021).

---

## Systems Engineering

### Python multiprocessing (mỗi tuyến 1 process)
- **traffic** (`backend/app/services/road_services/AnalyzeOnRoadForMultiProcessing.py`) — https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/AnalyzeOnRoadForMultiProcessing.py
  - `multiprocessing.Process(target=run_road_worker)` mỗi road.
  - Shared: `Manager().dict()` cho `info_dict`, `frame_dict`, `violations_list`, `control_dict`.
  - **Vì sao không thread:** GIL → OpenCV + YOLO là CPU-bound đối với phần Python → threading không parallel thật.
  - **Trade-off:** Manager là RPC socket nội bộ, không zero-copy. Scale > 5 tuyến 1080p sẽ nghẽn → giải: `multiprocessing.shared_memory` (Python 3.8+).

### Idempotent upsert (event_id key)
- **traffic** (`backend/app/services/road_services/ViolationAnalyzer.py` dòng 229-268) — `event_id = road:vehicle_id:frame:type` → `find_one({event_id})` → insert or update. Chống ghi trùng vi phạm cho cùng xe qua nhiều frame.

### Background worker + queue (không chặn hot path)
- **traffic** — OCR queue trong thread riêng (`_ocr_worker_loop` dòng 310-355), `violation_sync_loop` tick 3s ghi Mongo async. https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/violation_sync.py

### Hot-reload config không restart
- **traffic** (`backend/app/services/road_services/road_setup_runtime.py` dòng 372-399) — https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/road_setup_runtime.py
  - User PUT `/road-setups/{road}` → POST reload → set flag trong `control_dict` → process con hot-reload đầu vòng loop tiếp theo. Không restart backend.

### Race condition awareness (nói thẳng khi phỏng vấn)
- **badminton** — Check-then-write không lock ở overlap detection → 2 request concurrent có thể double-book. Giải production: PG `EXCLUDE` constraint hoặc `SELECT FOR UPDATE`.

---

## Databases

### PostgreSQL + SQLAlchemy 2.0 (async, Mapped/mapped_column)
- **badminton** — https://github.com/quyda2004/AI_badminton/blob/main/backend/models/
  - Models: User, Court, Booking, Payment.
  - Tiền dùng `Numeric(10, 2)`.
  - `server_default=func.now()` cho `created_at`.
- **ai-coworker** — async + asyncpg.

### MongoDB + Motor (async)
- **traffic** — Beanie 2.1 ODM. Collections: users, token_llm, chat_messages, violations, road_setups, traffic_stats.
- **badminton** — chat_history (`user_id`, `session_id`, `role`, `content`, `created_at`).
- **ai-coworker** — session state + chat.

### DB polyglot (RDBMS + NoSQL đúng use case)
- **badminton** — Postgres cho nghiệp vụ (FK, tiền, ràng buộc), Mongo cho chat (phi cấu trúc, ghi nhiều).
- **ai-coworker** — Graceful fallback: PG fail → hardcoded prompt, FAISS fail → "(No documents)", Mongo fail → memory only.

---

## DevOps / Deployment

### Docker Compose (5-service architecture)
- **badminton** — https://github.com/quyda2004/AI_badminton/blob/main/docker-compose.yml
  - `nginx` (80), `backend` (8000), `ai` (8001), `postgres` (5432), `mongodb` (27017).

### nginx API Gateway (strip prefix, proxy split)
- **badminton** — https://github.com/quyda2004/AI_badminton/blob/main/nginx/nginx.conf
  - `/api/*` → strip `/api` → `backend:8000`.
  - `/ai/*` → giữ path → `ai:8001`.
  - `/` → serve SPA static.

### Tách data khỏi source qua GitHub raw
- **portfolio** — Fetch `https://raw.githubusercontent.com/<user>/portfolio-knowledge-base/main/<path>` + cache Redis/in-memory TTL. Optional webhook invalidate.

---

## Testing

### pytest với reset + seed fixture
- **badminton** (`tests/conftest.py`) — https://github.com/quyda2004/AI_badminton/tree/main/tests
  - Reset + seed sạch trước mỗi run.
  - Coverage: auth (health, register, login, /me), courts (list, create, availability), bookings (create/get/cancel/reschedule + edge case 409/400).

### pytest-asyncio
- **ai-coworker** — https://github.com/quyda2004/AI_coworker/tree/main/tests
  - test_agents (persona registry), test_engine (state/director/safety), test_api (endpoint).

### Deterministic-only testing (biết giới hạn)
- Cả **badminton**, **ai-coworker**, **traffic** đều tự công khai: chưa cover LLM output → cần LLM-as-a-judge hoặc human eval trong future.

---

## Frontend

### React 19 + Vite 7 + TypeScript + shadcn/ui + TailwindCSS 4
- **traffic** — https://github.com/quyda2004/check/blob/main/frontend/package.json
  - `useTrafficStore.tsx` custom state (không dùng Zustand/Redux) với ring buffer 60 điểm cho line chart mật độ theo thời gian.
  - framer-motion animation, react-router-dom 7, lucide-react icon.

### WebSocket client (JPEG base64 stream)
- **traffic** — `ws.onmessage = e => setSrc(e.data)`. Compromise: đủ nhanh cho 9-10 FPS, dễ debug bằng browser devtools, code frontend đơn giản (chọn thay MJPEG vì backpressure control, thay WebRTC vì overkill).

---

## Data Engineering

### Custom dataset annotation với CVAT
- **traffic** — biển số 270 ảnh, Jaccard 99.95%, Dice 99.99% (không dùng Cohen's Kappa vì chỉ 1 class).

### Data augmentation (Mosaic + full stack)
- **traffic** — hue ±25°, exposure ±25%, noise 5%, shear ±15° ngang/dọc, saturation ±25%, blur ≤2.5px. Paper [1] báo cáo augmentation đẩy dataset 1.142 → 3.082 ảnh, mAP YOLOv8l 0.898 → 0.909.

### Keyword-based text extraction
- **ocr-financial-report** — Lọc theo "giám đốc", "CEO"... lấy ±150 từ trước/sau keyword → merge đoạn trùng → keep phần không trùng.

### Seed data + placeholder image
- **traffic** (`backend/seed.py`) — https://github.com/quyda2004/check/blob/main/backend/seed.py
  - TrafficStat: mật độ 6 khung giờ × 2 tuyến (hệ số 1.0 và 0.75).
  - 9 violation mẫu với biển số hard-code, `final_status=CHO_KET_LUAN`.
  - Ảnh placeholder JPEG chứa text mô tả → UI demo được không cần analyzer.

---

## Engineering Mindset (soft skill)

### Trung thực số liệu paper vs tự đo
- **traffic** — Trong `bao_cao_do_an.md` mục "Lời cam đoan" nhóm phân biệt rõ chỉ số paper (kế thừa vì cùng dataset + cùng pipeline) vs FPS/latency/RAM (tự đo trên hệ tích hợp). Không tuyên bố sai số lên hội đồng.

### Tự công khai limitations
- Cả 5 dự án đều có phần **"Nói thẳng — giới hạn hiện tại"** — hết mục nào cũng nêu điểm yếu để reviewer/phỏng vấn không phải tự moi ra.

### Task-based evaluation (né đánh giá cấu trúc)
- **ocr-financial-report** — Chọn extract field cụ thể + evidence + confidence thay vì tái dựng bảng gốc → né phải chấm TEDS/GriTS trên 9000 file, hợp mục tiêu lấy thông tin.
