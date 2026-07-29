# Portfolio Chatbot — Full Index

> Dự án này = chính chatbot đang xây. 2 file: `01` là **design** kiến trúc
> retrieval, `02` là **thực tế đã build** (backend + Docker + booking flow).

| File | Nội dung | Preview |
|---|---|---|
| `01-kien-truc-retrieval.md` | Tách data khỏi source qua GitHub raw + folder architecture chung 6 dự án + flow 3 lần gọi LLM (thô→tinh) + prompt caching + tool function design + guardrail | Repo `portfolio-knowledge-base` riêng; index.json + meta-all.md nạp thẳng; list_candidate_sections + get_section; cache_control ephemeral; max 5 tool call/câu hỏi |
| `02-backend-va-deployment.md` | Backend FastAPI monolith (main.py 830 dòng + github_fetcher.py) + Docker Compose 2 container (nginx + backend) + agentic loop max 6 vòng + booking Cal.com với OTP email + reload 3 tầng + khác biệt so với design gốc | 2 image quocan071/portfolio-{backend,nginx}; nginx ẩn ANTHROPIC_API_KEY; SSE stream `/api/chat`; OTP 5 phút + verified window 120s; webhook GitHub HMAC auto-reload; periodic refresh 15 phút |
