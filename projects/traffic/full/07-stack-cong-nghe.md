# Traffic — Stack Công Nghệ & Biến Môi Trường

## Stack công nghệ

**Source:** https://github.com/quyda2004/check/blob/main/backend/requirements_cpu.txt · https://github.com/quyda2004/check/blob/main/backend/requirements_gpu.txt · https://github.com/quyda2004/check/blob/main/frontend/package.json

| Layer | Công nghệ |
|---|---|
| Backend | FastAPI + Uvicorn |
| ODM | Beanie 2.1 (async, trên Motor 3.7) |
| DB chính | MongoDB (Motor 3.7 + PyMongo 4.11) |
| Auth | python-jose (JWT HS256) + passlib + bcrypt 4.0 |
| Vision core | Ultralytics YOLOv8 + OpenVINO (INT8 quantized) |
| Tracking | ByteTrack (qua Ultralytics) |
| Vision phụ trợ | OpenCV, cvzone, Shapely (polygon), lap (Hungarian), gdown |
| OCR biển số | EasyOCR + Gemini (fallback prompt-based OCR) |
| Detect biển số | Roboflow hosted model |
| Detect mũ bảo hiểm | Roboflow hosted model |
| LLM (chatbot + OCR) | `langchain-google-genai` (Gemini 2.5 Flash) |
| Agent framework | LangGraph `create_react_agent` (single-agent + ReAct) |
| Realtime | WebSocket native của FastAPI (4 kênh) |
| Multiprocessing | `multiprocessing.Process` + `Manager()` shared dict/list |
| Bot | `python-telegram-bot` → gọi `/chat_no_auth` |
| Frontend | React 19 + Vite 7 + TypeScript |
| UI | shadcn/ui (Radix primitives) + TailwindCSS 4 + lucide-react |
| Animation | framer-motion |
| Router | react-router-dom 7 |
| State | React Context + `useTrafficStore` (custom, không dùng Zustand/Redux) |
| Testing | pytest (backend), ESLint (frontend) |
| Legacy (không dùng runtime) | Alembic + SQLAlchemy scaffolding cho Postgres |

**Biến môi trường chính** (`backend/.env`): `MONGODB_URL`, `JWT_SECRET_KEY`, `JWT_ALGORITHM=HS256`, `GEMINI_API_KEY` (hoặc `GOOGLE_API_KEY`), `GEMINI_MODEL_NAME=gemini-2.5-flash`, `UNIFIED_ROAD_MODEL_PATH` (đường dẫn OpenVINO model), `DEVICE=cpu`.
