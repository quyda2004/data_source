# AI Co-Worker — RAG per-agent (FAISS) & Function Calling scaffolding

## 1. RAG theo từng agent

**Source:** [app/engine/agents/base_agent.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/agents/base_agent.py) · [app/knowledge/retriever.py](https://github.com/quyda2004/AI_coworker/blob/main/app/knowledge/retriever.py) · [app/db/vector/faiss_store.py](https://github.com/quyda2004/AI_coworker/blob/main/app/db/vector/faiss_store.py) · [app/knowledge/ingest.py](https://github.com/quyda2004/AI_coworker/blob/main/app/knowledge/ingest.py)

Mỗi agent có **knowledge base riêng** qua FAISS, để chỉ lấy đúng context liên quan:
- CEO → context về Group DNA, brand autonomy.
- CHRO → context về competency framework (VEPT).
- Regional Manager → context về rollout châu Âu.

**Chi tiết ingest:**
- Đọc `gucci_2.0.txt` → parse thành các SECTION.
- Chunk **500 ký tự**, **overlap 80**.
- Gắn metadata (`role_access`, `topic`).
- Build **4 FAISS index riêng**: `ceo/`, `chro/`, `regional/`, `shared/`.

**Cách agent dùng:**
- Mỗi agent **không đưa thẳng đáp án** mà đề xuất hướng đi và đánh giá xem hướng của user có ổn với định hướng không.
- Trong prompt mỗi agent còn có rule: nếu bị hỏi sai lĩnh vực (ví dụ CEO bị hỏi về HR hoặc vùng miền) thì **chuyển thẳng user sang agent đúng**, không trả lời bịa.

---

## 2. Function Calling — trạng thái thật

**Source:** [app/engine/tools.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/tools.py) · [app/engine/agents/base_agent.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/agents/base_agent.py)

Dự án **có định nghĩa 5 business tool** trong `app/engine/tools.py`:

| Tool | Hàm | Chức năng |
|---|---|---|
| KPI Calculator | `calculate_program_kpis` | Ước lượng cải thiện KPI cho chương trình lãnh đạo |
| A/B Simulator | `simulate_ab_scenarios` | So sánh 2 kịch bản rollout trên 5 chiều |
| Competency Lookup | `lookup_competency_framework` | Tra cứu khung năng lực VEPT |
| Resource Estimator | `estimate_resources` | Ước lượng budget/timeline/headcount |
| Regional Data | `get_regional_hr_data` | Lấy dữ liệu HR theo vùng |

Có sẵn `TOOL_REGISTRY` (mô tả + parameters) và hàm dispatch `invoke_tool(tool_name, **kwargs)`.

> **Nói thẳng cho đúng:** Các tool này **CHƯA được gắn thật vào LLM** dưới dạng native function calling. Agent chỉ gọi `self.llm.ainvoke(prompt)` với một prompt text thuần — **không** có `bind_tools`, **không** xử lý `tool_calls` trả về. Tức là tool tồn tại như một khung sẵn sàng, **gọi được bằng code** nhưng **LLM không tự quyết định gọi**. Đây là phần scaffolding để mở rộng sau, chưa phải function calling hoàn chỉnh.

**Cách hoàn thiện (chưa làm):**
- Với LangChain: dùng `llm.bind_tools([...])` để expose 5 hàm cho Gemini.
- Xử lý `tool_calls` trong response → dispatch qua `invoke_tool(name, **args)` → nhét kết quả lại làm ToolMessage.
- Hoặc dùng LangGraph `create_react_agent` với `tools=[...]`.
