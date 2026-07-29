# Portfolio Chatbot — Backend & Deployment (thực tế đã triển khai)

> File này bổ sung `01-kien-truc-retrieval.md`. Bản 01 mô tả **thiết kế**
> retrieval; bản 02 này ghi lại **những gì đã build thật** trong repo backend
> (FastAPI + Docker Compose + nginx). Có vài chỗ triển khai đi khác design gốc —
> ghi nhận rõ ở mục 6 (khác biệt so với design gốc).

---

## 1. Kiến trúc runtime — 2 container Docker

Deploy bằng **Docker Compose**, 2 service:

```
docker-compose.yml
├── nginx      (image quocan071/portfolio-nginx:latest, port 80)
│   ├── serve static: portfolio_lequocan.html + mat_quocan.jpg
│   └── proxy_pass /api/* → http://backend:8000
│       (proxy_buffering off, chunked_transfer_encoding on → hỗ trợ SSE streaming)
│
└── backend    (image quocan071/portfolio-backend:latest, KHÔNG expose port)
    ├── FastAPI + uvicorn
    ├── main.py (~830 dòng, monolith có chủ ý)
    └── github_fetcher.py (in-memory cache TTL 10 phút)
```

**Vì sao nginx đứng trước:**
- Ẩn `ANTHROPIC_API_KEY` server-side — browser chỉ gọi `/api/chat`, key nằm ở
  backend container, không lộ ra client.
- Serve HTML tĩnh nhanh (không đi qua Python).
- Không cần cấu hình CORS phức tạp (frontend + backend cùng origin).

**Vì sao backend không expose port:** chỉ nginx cần cổng 80; backend nói chuyện
với nginx qua network nội bộ của Docker Compose (`http://backend:8000`), giảm
attack surface.

---

## 2. Backend `main.py` — cấu trúc file

Cố ý để **1 file monolith** (~830 dòng) thay vì tách nhiều module — dự án nhỏ,
không cần over-engineer. Chia theo comment header rõ ràng:

```
main.py
├── ENV + load_dotenv           (ANTHROPIC_API_KEY, CAL_API_KEY, GMAIL_*, ...)
├── STATIC PROMPT               (behavior + tool rules, cứng trong code)
├── STATIC KNOWLEDGE build      (_build_static_knowledge — fetch từ GitHub raw)
├── APP + STARTUP               (FastAPI init, startup hook, periodic refresh)
├── EMAIL validation + OTP      (regex + storage + send OTP qua Gmail SMTP)
├── CAL.COM helpers             (_cal_get / _cal_post)
├── TOOLS (schema Claude)       (11 tool: 6 booking + 5 knowledge)
├── /api/chat                   (agentic loop max 6 vòng, SSE stream cuối)
├── /api/send-otp
├── /admin/reload-knowledge
├── /api/github-webhook         (HMAC verify + async reload)
├── /health
└── / và /mat_quocan.jpg        (serve static — dùng khi chạy trực tiếp,
                                  production nginx serve hộ)
```

**`github_fetcher.py`** tách riêng vì có state (cache dict) — không muốn lẫn với
main.py.

---

## 3. Static knowledge — build lúc startup

Khác design gốc (bản 01 nói 4 file tĩnh), thực tế nạp **3 file** vào system
prompt:

```python
# _build_static_knowledge()
personal, booking_flow, index_json = await fetch_many([
    "personal.md",       # thông tin cá nhân, học vấn, contact
    "booking-flow.md",   # quy tắc booking meeting (giờ hành chính, timezone, OTP...)
    "index.json",        # danh mục 6 dự án
])
```

**Không nạp `meta-all.md` thẳng** như design gốc — thay bằng tool
`get_project_meta(project_id)` gọi khi user hỏi cụ thể 1 dự án. Lý do: nạp cả 6
tóm tắt (~3-4KB) vào system prompt lãng phí cho những câu chỉ hỏi 1 dự án; để
LLM tự gọi tool khi cần thì đúng chi phí hơn.

**Không nạp `skill-detail.md`** — LLM tự suy được skill từ `personal.md` +
tag trong `index.json`. Nếu user hỏi sâu skill, gọi `get_project_chunk` để lấy
chi tiết.

---

## 4. Reload knowledge — 3 tầng

Có 3 cơ chế refresh chồng lên nhau, đi từ nhanh nhất tới an toàn nhất:

**Tầng 1 — GitHub Webhook (`/api/github-webhook`):**
- Instant. Push lên repo `quyda2004/data_source` → webhook bắn về backend →
  xác thực HMAC SHA256 (`GITHUB_WEBHOOK_SECRET`) → `asyncio.create_task` reload
  nền, trả 200 ngay cho GitHub (GitHub retry nếu response > 10s).
- Bỏ qua event khác `push`; `ping` trả pong.

**Tầng 2 — Periodic refresh (background loop):**
- Chạy nền mỗi **15 phút**, so sánh nội dung mới với cũ, chỉ log khi có thay
  đổi thật. Chống miss webhook.

**Tầng 3 — Manual (`POST /admin/reload-knowledge?key=...&wipe_cache=true`):**
- Có `RELOAD_KEY` guard. `wipe_cache=True` (default) xóa cache
  `github_fetcher` trước khi build lại — force fetch fresh, không đợi TTL 10
  phút.
- Dùng khi debug hoặc push xong muốn kiểm tra ngay.

---

## 5. Agentic loop — `/api/chat`

Không dùng SDK Anthropic — gọi thẳng REST `POST /v1/messages` bằng `httpx` cho
gọn. Loop tối đa **6 vòng** (guard) cho mỗi câu hỏi user:

```
loop for _ in range(6):
    resp = POST api.anthropic.com/v1/messages
           model:    claude-haiku-4-5-20251001
           system:   [_STATIC_KNOWLEDGE (cache_control ephemeral), NOW_STR]
           tools:    TOOLS  (11 tool)
           messages: history

    if stop_reason == "tool_use":
        chạy tool, append tool_result, tiếp vòng lặp
    else:
        stream text ra SSE, return
```

**Cache-friendly:** `_STATIC_KNOWLEDGE` gắn `cache_control: {type: "ephemeral"}`
→ từ vòng 2 trở đi phần này tính giá cache-read (rẻ ~10x). Block thời gian
tách riêng (không cache) để luôn có "now" mới.

**History cắt bớt:** `_safe_truncate(messages, max_len=10)` giữ 10 message gần
nhất để không nhồi context quá dài (mỗi vòng lặp phải gửi lại full history).

**Stream đầu ra qua SSE:** chỉ stream đoạn text cuối, không stream từng vòng
tool_use (user không cần xem tool nào được gọi). Nginx đã bật
`proxy_buffering off` để chunk chảy realtime.

---

## 6. Booking flow — Cal.com + OTP email

Chatbot không chỉ trả lời — có thể **đặt/hủy/dời lịch meeting thật** trên
Cal.com của Le Quoc An. Anti-abuse dùng OTP email 6 số:

```
User: "đặt lịch 10h sáng mai"
  ↓
LLM: get_available_slots(start=2026-07-30, end=2026-07-30)
LLM: (thấy 10:00 trống) hỏi user tên + email
  ↓
User: nhập email
LLM: gọi frontend UI → user click "Gửi OTP"
  ↓
POST /api/send-otp {email}
  → gen code 6 số, lưu _otp_store[email] TTL 5 phút
  → gửi Gmail SMTP (starttls port 587, app password)
  ↓
User: nhập code
LLM: verify_otp(email, code)
  → khớp → _otp_verified[email] = now, hiệu lực 120s
  ↓
LLM: create_booking(start_time, name, email)
  → SERVER enforce: _otp_verified[email] phải có trong 120s → nếu không, từ chối
  → POST cal.com/v2/bookings
  ↓
Đặt xong → email confirm silent (không raise nếu fail)
```

**In-memory OTP store** (`_otp_store`, `_otp_verified`) — 2 dict Python, có
cleanup mỗi lần gọi. Không dùng Redis vì chỉ 1 instance backend, TTL ngắn (5
phút OTP + 120s verified window), restart mất OTP dở dang là chấp nhận được.

**Tools booking (6 cái):** `get_available_slots`, `create_booking`,
`cancel_booking`, `reschedule_booking`, `get_booking`, `verify_otp`.

**Tools knowledge (5 cái):** `get_project_meta`, `get_project_chunk`,
`list_project_chunks`, `get_skills_map`, `get_skill_detail` — LLM chủ động gọi
khi cần đào sâu.

---

## 7. Khác biệt so với design gốc (`01-kien-truc-retrieval.md`)

Ghi lại rõ ràng — design là kim chỉ nam, thực tế đi khác vài chỗ do trade-off
khi build:

| Design gốc | Thực tế | Lý do đổi |
|---|---|---|
| Repo tên `portfolio-knowledge-base` | Repo `quyda2004/data_source` | Đặt tên gọn hơn khi tạo |
| Cache Redis hoặc in-memory | In-memory dict, TTL 10 phút | Chỉ 1 instance backend, đủ dùng |
| 4 file tĩnh nạp system prompt | 3 file (personal + booking-flow + index.json) | `meta-all.md` chuyển thành tool call — tiết kiệm token cho câu ngắn |
| Flow cố định 3 lần gọi LLM (`list_candidate_sections` + `get_section`) | Agentic loop max 6 vòng, tool tự do (`get_project_meta` / `get_project_chunk`) | Câu phức tạp có thể cần > 3 lần; câu đơn giản 1 lần là xong |
| Tool nhận `project_ids: list[str]` | Tool nhận `project_id: str` (1 dự án/lần) | Đơn giản hơn, LLM tự gọi song song nếu cần |
| Guardrail max 5 tool call | Loop hard-cap 6 vòng gọi API | Cùng ý tưởng, khác con số |

**Chưa làm so với design gốc:**
- GitHub Webhook — **đã làm** (mục 4 tầng 1).
- Prompt caching — **đã làm** (`cache_control: ephemeral`).
- Reload API — **đã làm** (`/admin/reload-knowledge`).

---

## 8. Deployment & CI

**Local dev / test:**
```bash
docker compose up --build -d       # build + chạy nền
docker compose logs -f backend     # xem log
docker compose down                # tắt
```

**Push image lên Docker Hub:** script `build-push.ps1` build 2 image
(`quocan071/portfolio-backend`, `quocan071/portfolio-nginx`) rồi push. Server
production pull về chạy — không build trên server để giữ server nhẹ.

**Env vars (`.env` root repo, backend load bằng `python-dotenv`):**
- `ANTHROPIC_API_KEY` (required — main.py crash startup nếu thiếu)
- `MODEL` (default `claude-haiku-4-5-20251001`)
- `CAL_API_KEY`, `CAL_EVENT_TYPE_ID` (booking)
- `GMAIL_SENDER`, `GMAIL_APP_PASSWORD` (OTP + confirm email)
- `RELOAD_KEY` (guard cho `/admin/reload-knowledge`)
- `GITHUB_WEBHOOK_SECRET` (verify HMAC webhook)
- `DISCORD_WEBHOOK_URL` (log câu hỏi user vào Discord, silent nếu thiếu)

**`.dockerignore` backend:** loại `__pycache__`, `*.pyc`, `.env` (env inject
qua compose `env_file`, không copy vào image).

---

## 9. Điểm yếu tự công khai

- **1 instance, in-memory state**: OTP, cache GitHub, static knowledge đều ở
  memory 1 container — không scale ngang được. Nếu 2 instance sau load balancer,
  OTP verified ở instance A không được thấy ở instance B. Chấp nhận vì
  portfolio cá nhân traffic thấp, restart nhanh.
- **CORS `allow_origins=["*"]`** cho gọn dev — production nên siết lại về
  origin thật.
- **Discord webhook log full câu hỏi user** — chưa mask PII. Portfolio public,
  ít nhạy cảm, nhưng nếu ai đó paste email/số điện thoại vào chat thì sẽ vào
  Discord.
- **Không có test** — main.py 830 dòng, chưa có test cho tool handler nào. Rủi
  ro cao nhất là `create_booking` (gọi API bên ngoài, tạo state thật) — nên
  ưu tiên test tool này trước.
- **OTP in-memory** — backend restart mất OTP dở. User phải request lại. Chấp
  nhận vì restart hiếm và TTL đã ngắn (5 phút).
