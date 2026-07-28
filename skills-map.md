# Skills Map — Bản rút gọn

> Bản đồ skill → dự án đã áp dụng. **Không kèm link evidence** để nhẹ khi nạp
> vào system prompt. Cần chi tiết + link → dùng `skill-detail.md`.

## Backend / API

- **FastAPI + Uvicorn** — traffic, badminton, ai-coworker
- **REST API design** — traffic (30+ endpoint), badminton (auth/courts/bookings/payments/admin), ai-coworker (chat/sessions)
- **WebSocket streaming** (native FastAPI) — traffic (4 kênh: frame, info, chat, admin resources)
- **JWT HS256 authentication** — traffic, badminton
- **Multi-service auth (chung SECRET_KEY)** — badminton (backend + ai-service tự decode JWT không cross-call)
- **Role-based access control** — traffic (`role_id=0` admin), badminton (`role="admin"`)

## AI / LLM Engineering

- **LangGraph ReAct agent** (`create_react_agent`) — traffic (single-agent 3 tool)
- **LangGraph multi-node graph** (7 node: Supervisor + Director + agents + Mentor + SafetyBlock) — ai-coworker
- **Native Gemini function calling** — badminton (6 tool)
- **Prompt engineering** — traffic (rules để LLM không bịa tên tuyến), badminton (rules không hỏi user ID nội bộ), ai-coworker (persona-based prompts với emotional adaptation)
- **Tool design cho AI** — badminton (`cancel_booking_by_info` giấu ID), traffic (tool ưu tiên live data, fallback DB)
- **MCP Server (FastMCP)** — badminton (wrap lại 6 tool cho Claude Desktop)
- **LLM-as-a-judge / consensus multi-model** — ocr-financial-report (3 LLM GPT-4o + Claude Haiku 4.5 + Claude 3.5 phải trùng tuyệt đối)
- **Token quota + cost control** — traffic (`token_quota.py`, 5000 token/user, `pre_model_hook` cắt history 2000 token)
- **Prompt caching (`cache_control: ephemeral`)** — portfolio (giữ chi phí thấp qua 3 lần gọi)

## RAG / Vector DB

- **FAISS per-agent isolated index** — ai-coworker (4 index: ceo/chro/regional/shared)
- **Chunking strategy** — ai-coworker (500 ký tự, overlap 80)
- **Metadata filtering** (`role_access`, `topic`) — ai-coworker
- **3-tier retrieval architecture** (meta → candidate preview → full section) — portfolio

## Computer Vision

- **YOLOv8 detection** — traffic (train 3 model: vehicle 6-class, helmet binary, plate 1-class)
- **Transfer learning** — traffic (theo pipeline paper Sadik/Aquitan/Moussaoui), ocr-financial-report (ResNet-18 phân loại 5 label hướng ảnh)
- **ByteTrack multi-object tracking** — traffic
- **Monocular 2D positioning (đo tốc độ)** — traffic (`meter_per_pixel` calibration)
- **OpenVINO INT8 quantization** — traffic (CPU inference 9-10 FPS)
- **EasyOCR + LLM Vision fallback** — traffic (Gemini fallback khi EasyOCR fail)
- **DeepSeek OCR** — ocr-financial-report (SOTA thời điểm dự án)
- **Table evaluation metrics** (TEDS, GriTS, Cell-level P/R/F1, Numeric cell accuracy) — ocr-financial-report (kiến thức nền)

## Signal Processing / Time-Series

- **1-D CNN nhận diện QRS** — af-detection-ecg (accuracy 99.92%)
- **Baseline wander removal** — af-detection-ecg (downsample 360→50Hz linear estimation)
- **K-means post-processing** — af-detection-ecg (chọn tâm cụm R-peak)
- **Feature engineering trên time-series** — af-detection-ecg (7 feature từ R-peak interval)
- **DeepCNN 24 lớp + 2 BiLSTM local/global** — af-detection-ecg
- **AAMI EC57 benchmark** — af-detection-ecg (Sensitivity ~98.1%, PPV ~97.6%)

## Systems Engineering

- **Multiprocessing (Python `multiprocessing.Process + Manager()`)** — traffic (mỗi tuyến 1 process con, GIL-free parallel)
- **Shared memory pattern (Manager dict/list)** — traffic (info_dict, frame_dict, violations_list, control_dict)
- **Hot-reload config không restart** — traffic (RoadSetup runtime + `control_dict` flag)
- **Idempotent upsert (event_id key)** — traffic (`road:vehicle:frame:type` chống ghi trùng vi phạm)
- **Background worker + queue** — traffic (OCR thread không chặn frame loop, `violation_sync` tick 3s)
- **Race condition awareness** — badminton (biết check-then-write + đề xuất `SELECT FOR UPDATE` hoặc PG exclusion constraint)

## Databases

- **PostgreSQL 16 + SQLAlchemy 2.0 (async)** — badminton, ai-coworker
- **MongoDB + Motor/Beanie** — traffic (Beanie 2.1 ODM), badminton (chat history), ai-coworker
- **DB polyglot (RDBMS + NoSQL)** — badminton (Postgres nghiệp vụ + Mongo chat), ai-coworker (Postgres + Mongo với graceful fallback)
- **Unique constraint + upsert** — traffic (event_id unique cho violation)

## DevOps / Deployment

- **Docker + Docker Compose** — badminton (5 khối), ai-coworker
- **nginx API Gateway** (strip prefix, proxy nhánh, serve SPA) — badminton
- **Fetch data qua GitHub raw (tách data khỏi source)** — portfolio
- **Cache TTL + webhook invalidation** — portfolio

## Testing

- **pytest fixture (reset + seed DB)** — badminton (`conftest.py` sạch trước mỗi run)
- **pytest-asyncio** — ai-coworker
- **Test coverage: auth, permission, business logic, edge cases** — badminton (test_auth/courts/bookings)
- **Deterministic-only testing (biết giới hạn không cover LLM output)** — ai-coworker, badminton (đề xuất LLM-as-a-judge cho future)

## Frontend

- **React 19 + Vite 7 + TypeScript** — traffic
- **shadcn/ui (Radix primitives) + TailwindCSS 4** — traffic
- **Custom state store (không dùng Zustand/Redux)** — traffic (`useTrafficStore` ring buffer 60 điểm)
- **WebSocket client + backpressure handling** — traffic
- **Vanilla HTML/CSS/JS SPA** — badminton, ai-coworker

## Data Engineering

- **PDF → Image conversion** — ocr-financial-report
- **Data augmentation** — traffic (mosaic + hue/exposure/noise/shear/saturation/blur theo paper [1])
- **Custom dataset annotation (CVAT)** — traffic (biển số, Jaccard 99.95%)
- **Keyword-based text extraction (±150 từ quanh keyword)** — ocr-financial-report
- **Seed data + placeholder image** — traffic (biểu đồ dashboard chạy được khi không có analyzer)

## Soft Skills / Engineering Mindset

- **Trung thực số liệu paper vs tự đo** — traffic (phân biệt mAP kế thừa paper vs FPS/latency/RAM nhóm tự đo trên hệ tích hợp)
- **Task-based evaluation** (né phải chấm cấu trúc bảng thô) — ocr-financial-report
- **Đề xuất hướng phát triển kèm độ dễ + độ ăn điểm** — traffic, badminton
- **Tự công khai limitations** — cả 5 dự án đều có phần "nói thẳng — giới hạn hiện tại"
