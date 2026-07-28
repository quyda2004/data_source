# Badminton — Full Index

**Repo:** https://github.com/quyda2004/AI_badminton

| File | Nội dung | Preview |
|---|---|---|
| `01-kien-truc-va-microservices.md` | Bối cảnh + kiến trúc 5 khối Docker Compose + overlap detection + 2 DB polyglot | nginx API Gateway; ai-service KHÔNG chạm PostgreSQL trực tiếp; công thức overlap; Postgres cho nghiệp vụ, Mongo cho chat |
| `02-function-calling-gemini.md` | JWT liên service + flow đặt sân + native function calling + API endpoints | Chung SECRET_KEY 2 service; 6 tool Gemini; `cancel_booking_by_info` giấu ID; endpoints backend + ai |
| `03-mcp-server.md` | Wrap 6 tool thành MCP Server (FastMCP) cho Claude Desktop | Chạy `python -m ai_service.mcp_server`; token rỗng nên tool auth fail (scaffolding chưa hoàn chỉnh) |
| `04-stack-va-database.md` | Chỗ INSERT thật + stack đầy đủ + test coverage + limitations | `db.commit()` mới INSERT thật; SQLAlchemy 2.0 + Motor; pytest reset+seed; 3 file test |
