# Portfolio Chatbot — Kiến Trúc Knowledge Base & Retrieval

> Dự án này = chính chatbot portfolio đang xây. File này mô tả lại kiến trúc
> đang triển khai (tham chiếu `CLAUDE.md` ở root repo backend).

---

## 1. Bối cảnh & mục tiêu

- Có 6 dự án làm knowledge base cho chatbot portfolio: `traffic`, `badminton`,
  `ai-coworker`, `ocr-financial-report`, `af-detection-ecg`, `portfolio` (chính
  dự án chatbot này).
- Nội dung một số dự án khá dài (traffic ~64KB), **không thể nhồi hết vào system
  prompt mỗi lần** — tốn token, chậm, không cần thiết vì phần lớn câu hỏi chỉ
  liên quan 1-2 phần nhỏ.
- **Mục tiêu 1:** tách data ra khỏi source code (repo GitHub riêng), để thêm/sửa
  dự án mới **không cần redeploy** app.
- **Mục tiêu 2:** kiến trúc retrieval nhiều tầng, chỉ load đúng phần cần thiết.
- **Mục tiêu 3:** cố định flow ở **3 lần gọi LLM** cho câu hỏi cần đào sâu, thiết
  kế sao cho rẻ nhất có thể trong 3 lần đó (không phải ép cứng số lần bằng logic).

---

## 2. Tách data khỏi source — repo GitHub riêng

Data (nội dung dự án) và code (backend chatbot) là **2 repo hoàn toàn tách biệt**,
vì tốc độ thay đổi khác nhau (data đổi liên tục, code ít đổi).

- Repo `portfolio-knowledge-base`: chỉ chứa các file `.md`/`.json` mô tả dự án.
- Repo backend/app: chỉ chứa logic gọi LLM, tool function, cache.

**App lấy data qua `raw.githubusercontent.com`** lúc runtime (không clone/build
kèm code):
```
GET https://raw.githubusercontent.com/<user>/portfolio-knowledge-base/main/<path>
```
Push data mới lên GitHub → lần fetch tiếp theo app tự thấy bản mới, **không
redeploy**. Có cache (Redis/in-memory, TTL vài chục phút) để tránh gọi GitHub
liên tục; optional gắn GitHub Webhook để tự invalidate cache khi có push mới.

---

## 3. Kiến trúc folder (đồng bộ cho tất cả 6 dự án)

Mỗi dự án — bất kể dài ngắn — dùng **chung 1 format**: có `full-index.md` (mục
lục) + folder `full/` (các file nội dung theo chủ đề). Dự án ngắn chỉ có 1-2
file trong `full/`, dự án dài (traffic) có nhiều file hơn — nhưng code xử lý
**giống hệt nhau cho mọi dự án**, không cần nhánh riêng.

```
knowledge-base/                     (repo GitHub riêng)
├── index.json                      # danh mục 6 dự án — LUÔN nạp, cache được
├── meta-all.md                     # tóm tắt gộp cả 6 dự án — LUÔN nạp, cache được
├── skills-map.md                   # bản rút gọn skill → project (không kèm link)
├── skill-detail.md                 # bản đầy đủ (có link evidence)
└── projects/
    ├── traffic/           (8 file trong full/)
    ├── badminton/         (4 file)
    ├── ai-coworker/       (3 file)
    ├── ocr-financial-report/  (2 file)
    ├── af-detection-ecg/  (2 file)
    └── portfolio/         (1 file)
```

### Nội dung từng loại file

**`index.json`** — danh mục 6 dự án, mỗi entry gồm: `id`, `name`, `one_liner`,
`tags`, `has_public_repo`, `repo` (nếu có), `detail_index` (path tới
`full-index.md` của dự án đó).

**`meta-all.md`** — tóm tắt ~300-500 từ/dự án, gộp cả 6 dự án vào 1 file, load
thẳng trong system prompt (không qua tool call).

**`full-index.md`** (riêng từng dự án) — bảng mục lục: file nào, nội dung gì,
preview ngắn (1 dòng) để LLM/server nhận diện khi nào cần mở file đó.

**`full/*.md`** — nội dung chi tiết theo chủ đề, mỗi file ~2-8KB.

---

## 4. Flow 3 lần gọi LLM (thiết kế "thô → tinh")

Vai trò của mỗi lần gọi khác nhau, không lần nào tốn ngang lần nào:

```
Call 1: [index.json + meta-all.md] (cached, tĩnh) + câu hỏi user
        → LLM đọc meta, thấy chưa đủ chi tiết
        → gọi tool: list_candidate_sections(project_id, question)

── Server xử lý (KHÔNG tính là gọi LLM) ──
   đọc projects/{id}/full-index.md
   → rank sơ bộ theo câu hỏi (keyword/embedding)
   → trả về TOP 2-3 dòng dạng {file, preview} — CHỈ preview, không phải nội
     dung đầy đủ (~50-100 token, rất rẻ)

Call 2: [context cũ] + list candidate (file + preview)
        → LLM tự chọn đúng file dựa trên preview thật
        → gọi tool: get_section(project_id, filename)

── Server xử lý ──
   đọc đúng 1 file trong projects/{id}/full/{filename} (~2-8KB)

Call 3: [context cũ] + nội dung file thật
        → LLM tổng hợp, trả lời cuối cùng cho user
```

**Vì sao thiết kế này rẻ hơn để server tự đoán 1 lần:**
- Lần 2 (bước "tìm") chỉ trả preview ngắn, không phải nội dung đầy đủ.
- LLM tự chọn file dựa trên preview thật → tỉ lệ chọn đúng cao hơn server tự
  `rank` mù, giảm rủi ro phải gọi thêm lần 4 để sửa sai.
- Lần 3 mới là lần duy nhất "tốn" (có nội dung thật), và luôn chỉ 1 file.

**Câu hỏi liên quan nhiều dự án cùng lúc:** tool `list_candidate_sections` /
`get_section` cần nhận `project_ids: list[str]` thay vì 1 id đơn, để không bị
đội thêm vòng gọi khi so sánh 2+ dự án.

**Số lần gọi không phải giới hạn cứng:** 3 lần là kỳ vọng ở trường hợp tốt
(happy path). Nên set 1 ngưỡng an toàn ở tầng hệ thống (VD **max 5 lần gọi
tool/câu hỏi**) để tránh loop vô hạn nếu có bug, chứ không ép cứng dừng ở 3 để
tránh trả lời thiếu/sai.

---

## 5. Prompt caching — giữ chi phí thấp qua nhiều lần gọi

Mỗi lần gọi tiếp theo phải gửi lại toàn bộ context cũ. Phần tĩnh (`index.json`,
`meta-all.md`, `skill-detail.md`) cần đánh dấu cache để không bị tính full giá
lặp lại ở call 2, call 3:

```javascript
system: [
  {
    type: "text",
    text: indexJsonContent + metaAllContent,
    cache_control: { type: "ephemeral" }
  }
]
```

Từ call 2 trở đi, phần này được tính giá cache-read (rẻ hơn input thường) thay
vì trả full giá mỗi lần.

---

## 6. Tool function cần implement (backend)

Hai hàm dùng chung cho cả 6 dự án — **không viết nhánh riêng theo độ dài dự án**:

```python
def list_candidate_sections(project_ids: list[str], question: str) -> list[dict]:
    """
    Đọc projects/{id}/full-index.md cho từng project_id.
    Rank sơ bộ theo câu hỏi (keyword hoặc embedding similarity).
    Trả về top 2-3 candidate: [{"project_id", "file", "preview"}, ...]
    KHÔNG trả nội dung đầy đủ.
    """

def get_section(project_id: str, filename: str) -> str:
    """
    Fetch nội dung 1 file cụ thể từ
    projects/{project_id}/full/{filename} (qua raw.githubusercontent.com,
    có cache).
    """
```

**Nguyên tắc:** gộp mọi nhu cầu "lấy chi tiết" vào ít tool nhất có thể, nhận
nhiều tham số (list project_ids) thay vì để LLM tự gọi tuần tự nhiều tool nhỏ lẻ.

---

## 7. Điểm khác biệt & Điểm yếu tự công khai

**Khác biệt so với RAG vector DB thông thường:**
- Không dùng embedding search cho mọi query — dùng **rank 2 tầng** (meta first → LLM
  chọn candidate → server fetch). Nhờ vậy: (1) tiết kiệm token vì không phải
  embed câu hỏi + retrieve nhiều chunk mỗi lần; (2) tận dụng khả năng LLM "hiểu
  câu hỏi" để chọn file, chính xác hơn cosine similarity thuần.
- Data là **file .md/.json được người viết cấu trúc chủ đích**, không phải chunk
  tự động từ text dump — nội dung theo chủ đề, `preview` đã tối ưu cho LLM chọn.

**Điểm yếu hiện tại (nói thẳng):**
- Chưa có cache invalidation qua Webhook → cache Redis chỉ dựa TTL, nội dung mới
  push GitHub có delay tối đa = TTL.
- Chưa có eval chất lượng LLM chọn candidate — không biết tỉ lệ chọn đúng file
  ở call 2 là bao nhiêu.
- Không có fallback nếu GitHub raw down (rất hiếm nhưng có thể xảy ra).
- Chưa xử lý case câu hỏi tổng hợp cực rộng ("kể tôi mọi thứ về Django trong 6 dự
  án") — có thể bị max tool call giới hạn.
