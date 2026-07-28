# AI Co-Worker — Bối cảnh, Kiến trúc Điều phối, LangGraph 7 node & Cơ chế cảm xúc

**Repo:** https://github.com/quyda2004/AI_coworker

## 1. Vì sao mình làm dự án này (bối cảnh)

**Source:** [README.md](https://github.com/quyda2004/AI_coworker/blob/main/README.md)

Hồi mình còn intern bên ITR, mình được các anh support rất nhiều. Nhưng có một vấn đề: vì mình còn là "trang giấy trắng" nên hỏi khá nhiều, mà mỗi lần hỏi thì các anh đang làm việc dễ bị ngắt mạch (break mood), còn mình thì cũng ngại hỏi dồn. Từ đó mình nghĩ: có cách nào tạo một "đồng nghiệp AI" để giải quyết đúng cái khoảng trống đó không — một chỗ để hỏi mà không sợ làm phiền ai, lại vẫn được định hướng đàng hoàng.

Sau đó mình tình cờ thấy đề bài training trainee của **Gucci Group**, thấy hợp nên lấy luôn làm bối cảnh mô phỏng. Người dùng đóng vai **OD Director**, trò chuyện với các đồng nghiệp AI (CEO, CHRO, Regional Manager) để hoàn thành các nhiệm vụ phát triển lãnh đạo.

**Bối cảnh mô phỏng:** Gucci Group HRM Talent & Leadership Development 2.0

| Nhân vật AI | Tên | Vai trò | Tính cách |
|---|---|---|---|
| CEO | Marco Bizzarri | Bảo vệ Group DNA, brand autonomy | Tầm nhìn, quyết đoán |
| CHRO | Elena Rossi | Dẫn dắt khung năng lực VEPT | Đồng cảm, có cấu trúc |
| Regional Manager | Sophie Dubois | Rollout châu Âu, vận hành | Thực tế, chi tiết |

---

## 2. Kiến trúc điều phối (Supervisor + Director)

**Source:** [app/engine/supervisor.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/supervisor.py) · [app/engine/director.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/director.py) · [app/engine/graph.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/graph.py) · [app/personas/prompts.py](https://github.com/quyda2004/AI_coworker/blob/main/app/personas/prompts.py)

Hệ thống dùng **LangGraph** với mô hình điều phối 2 lớp.

### Supervisor — định tuyến (router)
- Phân tích tin nhắn của user và quyết định ai sẽ trả lời tiếp theo.
- Output **chỉ một trong 4 giá trị**: `CEO`, `CHRO`, `RegionalManager`, `SafetyBlock`.
- Gọi Gemini với `temperature=0.0` (định tuyến ổn định) và `max_output_tokens=20` (chỉ cần một từ).
- Nếu user tự chọn agent từ UI thì Supervisor **tôn trọng lựa chọn**, không gọi LLM.
- **Không bao giờ** chọn Mentor — chỉ Director mới được kích hoạt Mentor. Nếu LLM lỡ trả về "Mentor", có `_fallback_keyword_route()` đếm keyword để chọn lại agent hợp lý.

> **Lưu ý chính xác:** `SafetyBlock` **không phải là một agent** — nó là một guardrail, không gọi LLM, không có persona, chỉ trả về một câu chặn cố định (`SAFETY_BLOCK_RESPONSE`).

### Director — giám sát tiến độ (chạy sau Supervisor, trước agent)
Director là một **node bên trong graph**, không phải agent tách rời. Luồng là `Supervisor → Director → agent`. Nhiệm vụ:
- Cập nhật `task_progress` (phát hiện task nào vừa hoàn thành).
- Cập nhật `sentiment_score` (cảm xúc user).
- Cập nhật `stuck_counter`.
- **Override sang Mentor** khi user bị kẹt hoặc khó chịu.

> **Đính chính bản chất:** Director **không** sửa việc Supervisor chọn nhầm agent, cũng không kéo agent về "đúng hướng nội dung". Việc duy nhất nó can thiệp là **chuyển sang Mentor** khi phát hiện user đang bí. Gọi nó là "lưới an toàn về tiến độ" thì đúng hơn là "bộ sửa định tuyến".

### Điều kiện kích hoạt Mentor
Mentor được gọi khi **một trong hai** điều kiện đúng:

1. **User bị stuck:** `stuck_counter >= 3` (3 lượt liên tiếp không có tiến bộ).
2. **User khó chịu:** `sentiment_score < 0.3`.

Ngoại lệ: nếu user tự chọn agent thủ công (`user_explicit_choice = True`), Director **không** override, tôn trọng lựa chọn của user.

Mentor chỉ cho **gợi ý ngắn (1–3 câu)** dựa trên tiến độ, **không đưa đáp án đầy đủ** — để user tự khám phá.

---

## 3. Cơ chế Stuck (chi tiết + trường hợp mình từng thắc mắc)

**Source:** [app/engine/director.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/director.py)

`detect_progress()` quét **5 tin nhắn gần nhất**, so với bảng `PROGRESS_SIGNALS` (8 task → danh sách keyword). Điểm mấu chốt:

- Nó **chỉ kiểm tra những task CHƯA hoàn thành** (`if not progress.get(task_key, False)`).
- Nếu tìm thấy keyword của một task chưa xong → đánh dấu task đó hoàn thành.
- `progress_made = (task_progress cũ != task_progress mới)` — tức là **chỉ tính là "có tiến bộ" khi có task MỚI vừa được đánh dấu xong**.
- Có tiến bộ → `stuck_counter` reset về 0. Không có tiến bộ → `stuck_counter += 1`.

**Trường hợp mình từng hỏi:** user đã làm xong 5 task rồi, trong 5 tin nhắn gần nhất **không** nhắc keyword của 3 task chưa xong, nhưng **có** nhắc keyword của task đã xong → stuck có +1 không?

→ **CÓ, stuck +1.** Vì `detect_progress` bỏ qua hoàn toàn các task đã hoàn thành, nên keyword của task đã xong **không** cứu được stuck. Không có task mới nào được đánh dấu → `progress_made = False` → `stuck_counter += 1`.

---

## 4. Cơ chế cảm xúc — 2 tầng khác nhau

**Source:** [app/engine/state.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/state.py) · [app/engine/director.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/director.py) · [app/engine/agents/ceo_agent.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/agents/ceo_agent.py) · [app/engine/agents/chro_agent.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/agents/chro_agent.py) · [app/engine/agents/regional_agent.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/agents/regional_agent.py)

Đây là chỗ dễ nói lẫn, cần phân biệt rõ:

### Tầng 1 — `sentiment_score` (cảm xúc của USER, dùng chung)
- Nằm trong State chung, khởi tạo cố định **0.5**.
- `detect_sentiment()` đếm từ tiêu cực vs tích cực trong tin nhắn user:
  - Negative signals: "i don't know", "confused", "stuck", "??"... → nếu neg > pos: **−0.15**
  - Positive signals: "great", "thanks", "i think", "my plan is"... → nếu pos > neg: **+0.10**
- Kẹp trong khoảng 0.0 – 1.0.
- Đây là chỉ số Director dùng để quyết định gọi Mentor.

### Tầng 2 — `EmotionalMemory` (cảm xúc của TỪNG AGENT với user)
Chỉ **3 agent** (CEO, CHRO, RegionalManager) mới có. Mentor và SafetyBlock **không** có. Mỗi agent giữ:
- `relationship_score`: khởi tạo **0.5**, kẹp 0.0 – 1.0.
- `tension_count`: đếm số lần user "chọc giận" agent.
- `last_topic`, `memorable_events` (giữ 5 sự kiện gần nhất).

**Cách điểm thay đổi mỗi lượt:**
- Lượt bình thường (không dính keyword tension): `relationship_score` **+0.05**.
- Lượt dính keyword tension: `relationship_score` **−0.1** và `tension_count += 1`.
  - CEO: "standardiz", "centraliz", "uniform"
  - CHRO: "skip anonymity", "no coaching", "ignore feedback"
  - Regional: "q3 rollout", "september", "rush", "same everywhere"

**"Nhớ dai":** `tension_count` là bộ đếm **độc lập** với `relationship_score`. Kể cả khi `relationship_score` đã lên 0.9, nếu trước đó user từng dính tension 5 lần thì agent vẫn "nhớ" đủ 5 lần đó và điều chỉnh giọng điệu (ví dụ `tension >= 3` → agent trở nên thẳng thừng hơn).

---

## 5. LangGraph — bao nhiêu node?

**Source:** [app/engine/graph.py](https://github.com/quyda2004/AI_coworker/blob/main/app/engine/graph.py)

Graph có đúng **7 node**:

1. `Supervisor` — định tuyến
2. `Director` — giám sát tiến độ / cảm xúc
3. `CEO` — agent
4. `CHRO` — agent
5. `RegionalManager` — agent
6. `Mentor` — agent gợi ý
7. `SafetyBlock` — guardrail

**Luồng:**
```
entry = Supervisor
Supervisor → Director (luôn đi qua Director)
Director → conditional edges → [CEO | CHRO | RegionalManager | Mentor | SafetyBlock]
Tất cả node đích → END   (one-turn per invoke: mỗi lần chỉ một agent trả lời rồi kết thúc)
```
