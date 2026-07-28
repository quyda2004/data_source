# AI Co-Worker — API Endpoints, Test, Stack & Graceful Fallback

## 1. API Endpoints

**Source:** https://github.com/quyda2004/AI_coworker/blob/main/app/api/routes/chat.py · https://github.com/quyda2004/AI_coworker/blob/main/app/api/routes/sessions.py · https://github.com/quyda2004/AI_coworker/blob/main/app/api/routes/health.py · https://github.com/quyda2004/AI_coworker/blob/main/app/main.py · https://github.com/quyda2004/AI_coworker/blob/main/app/api/middleware/safety.py

| Method | Path | Mô tả |
|---|---|---|
| GET | `/api/health` | Health check |
| POST | `/api/chat` | Gửi message, nhận response (route chính) |
| GET | `/api/chat/history/{session_id}` | Lấy lịch sử chat |
| GET | `/api/chat/state/{session_id}` | Xem session state (debug) |
| GET | `/api/chat/export/{session_id}` | Export chat ra file Markdown |
| GET | `/api/chat/sessions` | Liệt kê tất cả sessions |
| DELETE | `/api/chat/sessions/{session_id}` | Xóa session |
| POST | `/api/sessions/start?user_id=X` | Bắt đầu session mới |
| GET | `/api/sessions/{session_id}` | Lấy thông tin session |
| GET | `/` | Serve frontend hoặc JSON info |
| GET | `/docs` | Swagger UI |
| GET | `/redoc` | ReDoc |

**Luồng của `POST /api/chat`:**
1. `check_safety()` ở middleware (lớp chặn 1) → nếu không an toàn trả SafetyBlock luôn.
2. Load/create session state (in-memory cache + MongoDB).
3. Chạy LangGraph engine: Supervisor → Director → Agent → END.
4. Extract response, cập nhật state.
5. Background tasks (không block): lưu MongoDB + PostgreSQL.
6. Trả `ChatResponse` (agent, response, state_update, safety_flags, latency_ms).

---

## 2. Test & Đánh giá — trạng thái thật

**Source:** https://github.com/quyda2004/AI_coworker/tree/main/tests

Dự án **có unit test** (pytest + pytest-asyncio), thư mục `tests/`:
- `test_agents.py` — test persona registry.
- `test_engine.py` — test state, director, safety (ví dụ: sentiment giảm khi user confused, tăng khi user tự tin, progress detect từ keyword).
- `test_api.py` — test các endpoint.

> **Nói thẳng:** Dự án **chưa có đánh giá chất lượng đầu ra của LLM** (LLM output evaluation / benchmark). Những gì đang được test là **phần logic xác định (deterministic)**: định tuyến, cập nhật state, tính sentiment, các luật safety — những thứ có input/output rõ ràng nên unit test được.
>
> **Vì sao hợp lý:** chất lượng câu trả lời của LLM mang tính chủ quan, muốn đánh giá đàng hoàng thì cần human eval hoặc "LLM-as-a-judge" trên một bộ test case chuẩn — đây là hướng mở rộng tương lai. Ở giai đoạn prototype, ưu tiên là đảm bảo phần khung (routing/state/safety) chạy đúng và ổn định trước.

---

## 3. Stack công nghệ

**Source:** https://github.com/quyda2004/AI_coworker/blob/main/requirements.txt · https://github.com/quyda2004/AI_coworker/blob/main/app/engine/cache.py · https://github.com/quyda2004/AI_coworker/blob/main/app/personas/traits.py

| Layer | Công nghệ |
|---|---|
| Backend | FastAPI + Uvicorn |
| Multi-agent | LangGraph + LangChain |
| LLM | Google Gemini (gemma-3-1b-it) qua langchain-google-genai |
| Vector DB / RAG | FAISS (CPU) + sentence-transformers (all-MiniLM-L6-v2) |
| DB quan hệ | PostgreSQL (SQLAlchemy async + asyncpg) |
| DB phi cấu trúc | MongoDB (Motor async) |
| Validation | Pydantic + pydantic-settings |
| Frontend | Vanilla HTML/CSS/JS |
| Container | Docker + Docker Compose |
| Testing | pytest + pytest-asyncio |

**Fallback graceful:** PostgreSQL fail → dùng hardcoded prompts. FAISS fail → trả "(No documents)". MongoDB fail → chỉ dùng memory.
