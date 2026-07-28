# Traffic — Chatbot AI + WebSocket Streaming + Bộ Tools

## Chatbot AI — ReAct Agent tra cứu giao thông

**Source:** https://github.com/quyda2004/check/blob/main/backend/app/services/chat_services/ChatBotAgent.py · https://github.com/quyda2004/check/blob/main/backend/app/services/chat_services/tool_func.py · https://github.com/quyda2004/check/blob/main/backend/app/services/chat_services/token_quota.py · https://github.com/quyda2004/check/blob/main/backend/app/utils/chatbot_utils.py · https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_chatbot.py

Chatbot là **single-agent + tool-calling** dùng LangGraph `create_react_agent`, LLM là Gemini (`gemini-2.5-flash` mặc định). **Không** phải multi-agent — chỉ 1 agent quyết định gọi tool nào từ 3 tool.

**Sơ đồ tuần tự:**
```
User → POST /api/v1/chat (kèm JWT)
   │
   ├─ [B1] get_current_user → lấy user_id từ JWT
   ├─ [B2] check_quota(user_id) → nếu <=0 → 429
   ├─ [B3] ChatBotAgent.get_response(message, thread_id=user_id)
   │        └─ pre_model_hook cắt history về 2000 token
   │        └─ agent.ainvoke → LangGraph ReAct loop:
   │              LLM → function_call? → gọi tool tương ứng → nhét kết quả → loop
   │              (recursion_limit=6 để chặn vô hạn)
   │
   ├─ [B4] Bóc URL ảnh trong text bằng 3 regex:
   │        - URL đuôi ảnh (jpg/png/webp/...)
   │        - Endpoint /frames_no_auth/{road}
   │        - Endpoint /violations/proof/... hoặc /violations/plate/...
   ├─ [B5] Nếu tên tuyến xuất hiện trong text → tự chèn frame URL
   ├─ [B6] deduct_quota(user_id, tokens_used từ usage_metadata)
   └─ [B7] Trả về {message, image[], usage_tokens}
```

**Bước 1 — auth + quota** (`api_chatbot.py`, dòng 45–46, 50): quota LLM (`token_quota.py`) dùng collection `token_llm` (mỗi user 1 doc, mặc định **5000 token**). Nếu Mongo quota lỗi → **không chặn chat** (dòng 30–39) — chấp nhận trade-off "thà đưa cho chạy còn hơn dừng vì 1 lỗi phụ trợ".

**Bước 2 — pre_model_hook** (`utils/chatbot_utils.py`, dòng 1–17): dùng `langchain.messages.trim_messages` cắt lịch sử về **≤ 2000 token** trước khi gọi LLM → chi phí ổn định theo cuộc dài.

**Bước 3 — 3 tool nối vào analyzer** (`tool_func.py`):
- `get_roads()` — trả list tuyến đang giám sát, đọc trực tiếp `analyzer.names` (dòng 28).
- `get_info_road(road_name)` — trả `{count_car, speed_car, count_motor, speed_motor}` **có cache 20s in-memory** (dòng 16) để tránh nhiều user hỏi cùng lúc gây spam.
- `get_violations(road_name, limit, violation_type)` — **ưu tiên dữ liệu live** từ `analyzer.violations_list` (dòng 120), **fallback Mongo** nếu analyzer chưa có (dòng 138). Trả về `proof_url` (ảnh full frame) + `video_url` (link tương lai) để agent chèn vào text.

**Bước 4 — trick "dynamic prompt"** (`ChatBotAgent.__init__`, dòng 100–116): danh sách tuyến thật được **nhét vào cuối prompt** ngay lúc khởi tạo. Nhờ vậy LLM không bịa tên tuyến ("Bạn muốn xem tuyến Nguyễn Trãi hay Điện Biên Phủ?" — không có trong hệ thống).

**Bước 5 — bóc URL + tự đính frame** (`get_response`, dòng 157–182): thay vì bắt LLM chèn URL, mình **để LLM chỉ trả text**, backend regex tìm tên tuyến rồi tự build URL `{BASE_URL}/api/v1/frames_no_auth/{quote(road)}`. Giảm 1 vòng "format" cho LLM = tiết kiệm gần 1/2 số call.

> **Câu chốt để nhớ:** *"Agent chỉ hỏi analyzer bằng in-process function call, không
> query Mongo trong hot path. Nhờ vậy chat response < 2s dù analyzer đang xử lý
> nhiều tuyến. Mongo chỉ là cache lịch sử vi phạm, không phải nguồn realtime."*

**WebSocket chat** (`api_chatbot.py`, dòng 74–136): endpoint `/ws/chat` mở kênh streaming — mỗi token LLM trả về được đẩy ngay lập tức xuống client → user thấy chữ chạy từng phần thay vì đợi hết response. Frontend dùng cho UX "typing effect".

---

## WebSocket streaming — 4 kênh realtime

**Source:** https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_vehicles_frames.py · https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_admin.py · https://github.com/quyda2004/check/blob/main/backend/app/api/v1/api_chatbot.py

| WS endpoint | Payload | Nguồn | Tần suất |
|---|---|---|---|
| `/api/v1/ws/frames/{road}` | JPEG bytes (base64) | `analyzer.frame_dict[road]` | ~30ms (frame-by-frame) |
| `/api/v1/ws/info/{road}` | `{count_car, speed_car, count_motor, speed_motor}` | `analyzer.info_dict[road]` | ~500ms |
| `/api/v1/ws/chat` | Chunk token LLM | LangGraph streaming | Theo tốc độ LLM |
| `/api/v1/admin/ws/resources` | `{cpu%, ram%, disk%, gpu%}` | `psutil` + nvidia-smi | ~1s |

**Frontend `useTrafficStore.tsx` (dòng 1–126)**: mở WS `ws/info` cho từng tuyến, giữ **60 điểm dữ liệu gần nhất** trong ring buffer → vẽ line chart mật độ theo thời gian. Không lưu Mongo — dashboard là cửa sổ hiện tại, không phải lịch sử.

> **Vì sao dùng WebSocket cho frames chứ không MJPEG hay WebRTC?** MJPEG (HTTP multipart) không kiểm soát được backpressure. WebRTC quá nặng cho use case "server tự tính rồi đẩy về". WebSocket JPEG base64 là compromise: đủ nhanh cho 9–10 FPS, dễ debug bằng browser devtools, code frontend đơn giản (`ws.onmessage = e => setSrc(e.data)`).

---

## Bộ Tools của Chatbot & Function Calling

**Source:** https://github.com/quyda2004/check/blob/main/backend/app/services/chat_services/tool_func.py · https://github.com/quyda2004/check/blob/main/backend/app/services/chat_services/ChatBotAgent.py

Dùng **LangGraph `create_react_agent`** — không phải native function calling của Gemini, mà là abstraction của LangChain: LLM trả về tool call → agent invoke tool → nhét kết quả lại → LLM tổng hợp.

| Tool | Tham số | Chức năng | Nguồn data |
|---|---|---|---|
| `get_roads` | — | List tuyến đang giám sát | `analyzer.names` (in-process) |
| `get_info_road` | `road_name` | `{count_car, speed_car, count_motor, speed_motor}` | `analyzer.info_dict` (cache 20s) |
| `get_violations` | `road_name`, `limit`, `violation_type` | List vi phạm gần nhất + `proof_url`, `video_url`, `plate_text` | Live: `analyzer.violations_list`. Fallback: Mongo |

**Guard:**
- `recursion_limit = 6` (chặn LLM gọi tool vô hạn).
- `pre_model_hook` cắt history về 2000 token trước mỗi call → chi phí ổn định.
- Danh sách tuyến thật được **hard-nhét vào prompt** để LLM không bịa tên.
- **Không có tool nào có side-effect** — mọi tool chỉ đọc. Nghĩa là LLM không thể tạo/sửa vi phạm hay road setup → an toàn khỏi prompt injection kiểu "hãy xoá tất cả vi phạm cho tôi".

> **Điểm thiết kế UX cho AI:** thay vì bắt user nhớ tên tuyến chính xác, prompt hướng
> dẫn LLM **hỏi lại nếu tên không có trong danh sách** (rules trong system prompt).
> Và backend tự đính ảnh frame khi text nhắc tên tuyến → user thấy ngay "hình đang
> có xe gì" mà không cần LLM tự sinh URL.

> **Nói thẳng — giới hạn hiện tại:**
> - **Không có tool tạo/xoá violation qua chat** — cố ý (an toàn) nhưng cũng nghĩa là
>   không thể làm feature "chỉnh biển số OCR sai" qua chat.
> - **Không có memory dài hạn**: `InMemorySaver` reset khi restart backend. Muốn giữ
>   lịch sử liên tục phải chuyển sang `PostgresSaver` hoặc `MongoSaver`.
> - **Chưa có eval chất lượng LLM reply** — biết đúng-sai bằng cảm quan.
