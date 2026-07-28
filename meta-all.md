# Meta — Tóm tắt 6 dự án portfolio

> File này gộp tóm tắt ngắn (~300-500 từ/dự án) cho cả 6 dự án. Nạp thẳng vào
> system prompt cùng `index.json` để LLM có ngữ cảnh nền cho **mọi** câu hỏi,
> không phải gọi tool cho câu hỏi cơ bản.

---

## 1. Smart Traffic System (`traffic`)

**Repo:** https://github.com/quyda2004/check

**Bối cảnh:** Đồ án tốt nghiệp KHDL (ĐH KHTN – ĐHQG TP.HCM). Ghép nhiều mô-đun CV
+ AI + web + realtime thành 1 pipeline chạy được và hỏi được: detect xe → tracking
→ đo tốc độ → phát hiện vi phạm đèn đỏ → OCR biển số → cho ReAct Agent tra cứu
bằng tiếng Việt. **9-10 FPS trên CPU phổ thông**, song song nhiều tuyến đường.

**Kiến trúc:** 3 service — `backend` (FastAPI + multiprocessing analyzer),
`frontend` (React 19 + Vite 7), `mongodb`. **Không dùng API Gateway** cho gọn demo.
Nguyên tắc cốt lõi: **mỗi tuyến 1 process con** (Python multiprocessing) — analyzer
KHÔNG chạy trong request handler; endpoint chỉ đọc từ `Manager().dict()` shared
memory. 1000 client cùng xem 1 tuyến vẫn chỉ 1 process xử lý video.

**AI stack:** YOLOv8l 6-class (train theo pipeline paper Sadik et al.) → ánh xạ
Car/Motor, export **OpenVINO INT8**; ByteTrack cho tracking; monocular 2D
positioning đo tốc độ; EasyOCR + Gemini fallback đọc biển số; Roboflow hosted
cho helmet detection. Chatbot dùng **LangGraph `create_react_agent`** với 3 tool
(`get_roads`, `get_info_road`, `get_violations`) — đọc trực tiếp analyzer state,
không query Mongo trong hot path. `recursion_limit=6`, `pre_model_hook` cắt
history về 2000 token.

**Điểm nổi bật:** vi phạm có `event_id = road:vehicle:frame:type` duy nhất →
idempotent upsert; OCR queue trong thread riêng, không chặn frame loop; sync
Mongo mỗi 3s. Auth JWT HS256, WS truyền token qua query param. Realtime qua 4
kênh WebSocket (frame, info, chat streaming, admin resources).

**Điểm yếu tự công khai:** Manager()-based shared memory nghẽn khi scale > 5
tuyến 1080p (giải: `multiprocessing.shared_memory`). Chatbot dùng `InMemorySaver`,
reset khi restart (giải: MongoSaver). Alembic legacy còn sót từ nhánh SQL cũ.
Nhiều endpoint public do demo. Chưa có test cho ViolationAnalyzer.

---

## 2. BadmintonPro (`badminton`)

**Repo:** https://github.com/quyda2004/AI_badminton

**Bối cảnh:** Đặt sân cầu lông bằng hội thoại tiếng Việt tự nhiên thay vì form —
"Đặt sân 1 ngày mai lúc 8h" → hệ thống tự hiểu, tra, thực hiện. Muốn biến chatbot
chung chung thành AI agent **hành động thật trong hệ thống nghiệp vụ**.

**Kiến trúc microservice 5 khối:** `nginx` (API Gateway, strip `/api` cho backend,
giữ `/ai/*` cho ai-service), `backend` (FastAPI + SQLAlchemy), `ai` (FastAPI +
Gemini), `postgresql` (nghiệp vụ), `mongodb` (chat history). **Ai-service KHÔNG
truy cập thẳng PostgreSQL** — gọi ngược REST API backend qua HTTP kèm JWT của
user. Nhờ vậy business logic (check trùng giờ, tính giá, phân quyền) chỉ tồn tại
1 chỗ ở backend.

**Auth liên service:** cả 2 service dùng chung `SECRET_KEY` → ai-service tự decode
JWT cục bộ, không gọi backend hỏi. Hệ quả: AI **không có siêu quyền** — mọi thao
tác chạy với đúng quyền user đang login, backend enforce như thường.

**Function calling native Gemini:** 6 tool (`get_courts`, `check_availability`,
`create_booking`, `get_my_bookings`, `cancel_booking`, `cancel_booking_by_info`).
Tool `cancel_booking_by_info` cho hủy bằng tên sân + ngày + giờ (user không biết
`booking_id` nội bộ) — giấu chi tiết kỹ thuật. Có wrap thêm **MCP Server**
(FastMCP) để Claude Desktop dùng chung bộ tool.

**Business logic quan trọng:** overlap detection dùng công thức `start_cũ < end_mới
AND end_cũ > start_mới`, áp ở 3 chỗ (create/reschedule/availability).

**Điểm yếu:** race condition (check-then-write không khóa — có thể double-booking
khi concurrent; giải: PG exclusion constraint hoặc `SELECT FOR UPDATE`); vòng
lặp function calling chưa có `recursion_limit`; chưa có tool reschedule cho AI;
MCP tool token rỗng nên fail auth.

---

## 3. AI Co-Worker Engine (`ai-coworker`)

**Repo:** https://github.com/quyda2004/AI_coworker

**Bối cảnh:** Từ trải nghiệm intern hay ngại hỏi làm phiền anh senior → tạo "đồng
nghiệp AI" để hỏi mà không sợ làm phiền. Bối cảnh mô phỏng: Gucci Group HRM
Talent & Leadership Development 2.0 — user là OD Director, trò chuyện với 3 AI
persona (CEO Marco Bizzarri, CHRO Elena Rossi, Regional Manager Sophie Dubois).

**Kiến trúc điều phối 2 lớp (LangGraph, 7 node):** Supervisor định tuyến (chỉ
trả về CEO/CHRO/RegionalManager/SafetyBlock, `temperature=0.0`, max 20 token) →
Director giám sát tiến độ (update `task_progress`, `sentiment_score`,
`stuck_counter`) → agent trả lời. Director **chỉ can thiệp bằng cách override
sang Mentor** khi user bị kẹt (`stuck_counter >= 3`) hoặc khó chịu
(`sentiment_score < 0.3`). Không sửa được Supervisor chọn nhầm agent.

**Cảm xúc 2 tầng riêng biệt:** (1) `sentiment_score` chung của user (khởi tạo
0.5, đếm keyword neg/pos) — Director dùng; (2) `EmotionalMemory` từng agent
(`relationship_score` + `tension_count`) — mỗi agent "nhớ dai" khi bị chọc giận
lặp, giọng thẳng thừng hơn khi `tension >= 3`.

**RAG per-agent:** 4 FAISS index riêng (`ceo/`, `chro/`, `regional/`, `shared/`),
chunk 500 ký tự overlap 80. Mỗi agent không đưa đáp án mà đề xuất hướng đi + đánh
giá — bị hỏi sai lĩnh vực thì tự chuyển user sang agent đúng.

**Điểm cần nói thẳng:** 5 business tool trong `tools.py` **CHƯA gắn thật vào LLM**
dưới dạng function calling — chỉ dispatch được bằng code, LLM không tự quyết
định gọi (scaffolding chưa hoàn chỉnh). Chưa có LLM eval; test chỉ cover phần
deterministic (routing/state/safety).

---

## 4. OCR 9000 Báo cáo tài chính (`ocr-financial-report`)

**Bối cảnh:** 9000 file báo cáo tài chính công ty (dữ liệu nội bộ), định dạng
không đồng bộ (text/scan). OCR toàn bộ để chuẩn hóa input.

**Pipeline:** PDF → ảnh → **ResNet-18** phân loại 5 label hướng (transfer
learning, phát hiện & sửa 20.000+ ảnh sai hướng trong 500.000 ảnh) → **DeepSeek
OCR** (SOTA thời điểm dự án, xử lý tốt tiếng Việt + báo cáo tài chính) → extract
information bằng prompt có 2 cột phụ **Evidence** (trích text gốc) + **Confidence**.

**Bài toán đánh giá chất lượng khi không thể label thủ công 9000 file:** sinh 2
bộ dữ liệu con dựa trên điểm chung (thông tin giám đốc/HĐQT có ở mọi báo cáo
thường niên) → lọc theo keyword ("giám đốc", "CEO"...) lấy ±150 từ → merge đoạn
trùng → chạy cùng prompt trên **3 LLM khác nhau** (GPT-4o, Claude Haiku 4.5,
Claude 3.5). **Tiêu chí: cả 3 phải trùng tuyệt đối mới nhận**, không trùng →
human gán nhãn thủ công.

**Kết quả consensus:** Giám đốc 97.3%, HĐQT 98.6%. Phần sai chủ yếu do OCR chưa
extract tốt, không phải lỗi prompt/LLM.

**Kiến thức nền được vận dụng:** biết các metric đánh giá bảng chuyên biệt (TEDS
— tree edit distance, GriTS — grid similarity, Cell-level P/R/F1, Numeric cell
accuracy cho tài chính). Chọn **task-based evaluation** (field cụ thể + evidence)
để né việc phải đánh giá cấu trúc bảng thô — hợp với mục tiêu lấy thông tin
chứ không phải số hóa nguyên bảng.

---

## 5. AF Detection từ tín hiệu ECG (`af-detection-ecg`)

**Bối cảnh:** Công ty giao nghiên cứu phát hiện rung nhĩ (Atrial Fibrillation)
theo 2 hướng song song: **Device** (chạy on-device, ưu tiên nhanh + nhẹ) và
**Cloud** (server, model lớn hơn, tích hợp paper tham khảo). Bản chất: AF nhận
biết qua **loạn nhịp giữa các sóng QRS** — bình thường nhịp đều, bệnh không đều.

**Hướng 1 — Device (ML truyền thống):**
- Tín hiệu 360Hz → downsample 50Hz để ước lượng baseline linear (khung 4s) →
  trừ baseline mà không mất QRS (linear coi QRS là outlier) → normalize.
- **1-D CNN nhận diện QRS:** label 1 cho điểm ±40ms quanh R-peak, sample -100ms
  → +300ms. Hậu xử lý K-means (cửa sổ 80ms, >5 điểm label 1 → lấy tâm cụm).
- **Kết quả:** Accuracy QRS **99.92%** trên 22 người test (tách hoàn toàn khỏi
  train). Sau đó extract 7 feature từ R-peak → Random Forest/XGBoost phân loại
  AF/Normal: ~96.5% accuracy, F1 ~95.8%. Robust trên nhiều SNR ~90-98%.

**Hướng 2 — Cloud (Deep Learning end-to-end):**
- Tiền xử lý: tách nhiễu theo frequency band + median filter.
- **DeepCNN 24 lớp** (3 block × 8 conv, filter 32/64/128, mỗi block 1 pooling
  hệ số 4) + **2 lớp BiLSTM** (lớp 1 học local, lớp 2 học global).
- Train 150-200 epoch, batch 256-512, LR scheduler + early stopping.
- **Kết quả:** phân loại 4 class (Normal/AF/Other/Noisy) accuracy **~98.3%**,
  cross-dataset ~95-97%, đạt chuẩn **AAMI EC57** với Sensitivity ~98.1%,
  Positive Predictivity ~97.6%.

**Trade-off:** Device nhanh, nhẹ, có thể bỏ sót hình thái sóng phức tạp; Cloud
học đầy đủ hơn nhưng cần compute lớn, không on-device được.

**Paper tham khảo:** Šarlija et al. (ISPA 2017) cho CNN QRS; Cheng et al. (2021)
cho DCNN+BiLSTM 4-class.

---

## 6. Portfolio Chatbot (`portfolio`)

**Bối cảnh:** Chính dự án chatbot đang xây — trả lời câu hỏi về 5 dự án trên
bằng knowledge base có kiến trúc retrieval nhiều tầng.

**Kiến trúc data:** tách data khỏi source (repo `portfolio-knowledge-base` riêng),
app fetch qua `raw.githubusercontent.com` lúc runtime + cache Redis/in-memory
TTL vài chục phút → thêm/sửa dự án **không cần redeploy**. Optional gắn GitHub
Webhook để invalidate cache tự động.

**Kiến trúc folder đồng bộ:** mọi dự án dùng chung format `full-index.md` + folder
`full/*.md`. Dự án ngắn có 1-2 file, dài (traffic) có 8 file — code xử lý giống
nhau cho mọi dự án. `index.json` + `meta-all.md` + `skills-map.md` +
`skill-detail.md` là 4 file tĩnh nạp thẳng vào system prompt.

**Flow 3 lần gọi LLM (thô → tinh):** Call 1 với meta → LLM gọi tool
`list_candidate_sections(project_ids, question)` → server đọc `full-index.md`,
rank sơ bộ, trả top 2-3 candidate CHỈ với preview. Call 2 LLM chọn file → gọi
`get_section(project_id, filename)` → server fetch đúng 1 file. Call 3 LLM tổng
hợp trả lời cuối.

**Tối ưu chi phí:** prompt caching `cache_control: ephemeral` cho phần tĩnh
(`index.json + meta-all.md + skill-detail.md`) → call 2, 3 tính cache-read thay
vì full input price. Tool nhận `project_ids: list[str]` (không phải 1 id đơn) để
so sánh nhiều dự án không đội thêm vòng gọi.

**Guardrail:** không ép cứng dừng ở 3 lần — set max 5 tool call/câu hỏi ở tầng
hệ thống để tránh loop vô hạn khi bug, nhưng cho phép LLM gọi thêm khi thật sự
cần đào sâu.
