# AI Co-Worker — Full Index

**Repo:** https://github.com/quyda2004/AI_coworker

| File | Nội dung | Preview |
|---|---|---|
| `01-multi-agent-supervisor-director.md` | Bối cảnh + kiến trúc điều phối 2 lớp (Supervisor + Director) + LangGraph 7 node + cơ chế stuck + cảm xúc 2 tầng | Gucci Group HRM mô phỏng; Supervisor định tuyến 4 giá trị; Director override sang Mentor; sentiment_score chung + EmotionalMemory từng agent với tension_count |
| `02-rag-faiss-pipeline.md` | RAG per-agent (4 FAISS index) + 5 business tool (chưa gắn vào LLM) | Chunk 500/overlap 80; ceo/chro/regional/shared; TOOL_REGISTRY scaffolding chưa hoàn chỉnh function calling |
| `03-stack-va-trien-khai.md` | API endpoints + test coverage + full stack + graceful fallback | 12 endpoint chat/session; test_agents/engine/api; FastAPI + LangGraph + FAISS + Gemma-3-1b + fallback khi PG/FAISS/Mongo fail |
