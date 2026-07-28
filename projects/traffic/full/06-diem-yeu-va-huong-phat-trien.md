# Traffic — Điểm khác biệt, Điểm yếu & Hướng phát triển

## Điểm khác biệt so với chatbot thị trường + Hướng phát triển

**Source:** tổng hợp thiết kế (README + `services/chat_services/` + `services/road_services/`)

**Khác biệt thật sự (đã có):** dự án **không cố làm AI thông minh hơn Gemini**. Khác biệt nằm ở:

1. **Chatbot trả lời dữ liệu realtime, không phải kiến thức có sẵn.** Mọi câu trả lời đến từ tool đọc live analyzer — user hỏi lúc 8h thấy khác 20h vì thực sự khác.
2. **Chatbot tự đính ảnh bằng chứng vi phạm.** Không phải "kể chuyện" mà đưa link ảnh hoặc frame stream — user click vào xem được biển số thật, ảnh xe thật.
3. **Chatbot tôn trọng danh sách tuyến thật.** Không bịa tên đường, không đoán vi phạm — vì prompt được inject danh sách runtime.
4. **Pipeline vision + LLM song song, không chờ nhau.** Analyzer 9–10 FPS liên tục, LLM chỉ được gọi khi user hỏi. Frame stream không bị chậm vì đang chờ Gemini OCR.

**Điểm yếu hiện tại (nên tự nói ra khi phỏng vấn):**

- Không có memory hội thoại dài hạn (InMemorySaver reset khi restart).
- Không có eval LLM reply (không đo được lúc nào Gemini trả sai).
- Alembic legacy còn sót, dễ gây hiểu nhầm "dùng cả 2 DB".
- Chưa có xử lý race khi user PUT road-setup lúc analyzer đang đọc config đó.
- OCR fail thì event vẫn được lưu với `plate_text=""` — chưa có UI retry.
- WebSocket frame chưa auth ở production mode.
- ViolationAnalyzer không có unit test — logic crossing dễ regress.
- Nhiều endpoint public trong khi có dữ liệu nhạy cảm (biển số xe).
- Manager()-based shared memory sẽ nghẽn ở scale > 5 tuyến 1080p đồng thời.

**Hướng phát triển gợi ý (xếp theo độ dễ + độ ăn điểm):**

1. **Chuyển checkpointer sang MongoSaver** — 1 buổi làm, cuộc chat liên tục qua nhiều session, ăn điểm rõ "hiểu LangGraph sâu".
2. **Thêm tool cho AI: `get_hourly_stats(road, date)`** — đọc từ `traffic_stats` để user hỏi "hôm qua giờ nào đông nhất". Tận dụng seed data sẵn có.
3. **Alert Telegram khi có vi phạm mới** — mở rộng `bot_tele.py` gửi ảnh proof kèm biển số ngay khi `violation_sync` insert bản ghi mới. Ăn điểm "hệ thống có tác vụ chủ động".
4. **shared_memory + numpy zero-copy** — thay `Manager().dict()` bằng `multiprocessing.shared_memory` cho frame → giảm CPU serialize, mở đường scale nhiều tuyến 1080p. Tư duy production.
5. **Guardrails + audit log tool call** — log mọi tool call của LLM ra Mongo → truy vết được khi Gemini trả bậy.

> **Lời khuyên phỏng vấn:** chọn 1–2 hướng làm cho tới thay vì rải hết. Gợi ý mạnh
> nhất: *MongoSaver + alert Telegram* (dễ demo, thấy ngay hiệu ứng) + *shared_memory*
> (tư duy kỹ sư hệ thống, phân biệt được sinh viên đọc tutorial vs sinh viên hiểu OS).
