# Traffic — Full Index

> Mục lục các file chi tiết dự án **Smart Traffic System**. LLM đọc file này ở
> Call 2 (sau khi meta không đủ) để chọn file nào cần load.

**Repo:** https://github.com/quyda2004/check

| File | Nội dung | Preview |
|---|---|---|
| `01-boi-canh-va-kien-truc.md` | Bối cảnh đồ án tốt nghiệp + kiến trúc 3 khối tổng thể | Vì sao làm dự án; 3 service backend/frontend/mongodb; nguyên tắc cốt lõi analyzer chạy trong process con, endpoint chỉ đọc shared memory; auto-start lifecycle |
| `02-auth-phan-quyen.md` | JWT HS256 + role_id + WS token qua query param | OAuth2 password flow; `role_id=0` admin; WS `?token=`; endpoint `chat_no_auth` cho Telegram + demo |
| `03-video-pipeline-multiprocess.md` | Multiprocessing analyzer + train YOLO + phát hiện vi phạm | Mỗi tuyến 1 Process; 5 mô-đun CV (3 train YOLOv8 + ByteTrack + speed); OpenVINO INT8; event_id + OCR queue |
| `04-chatbot-agent.md` | ReAct Agent + 3 tool + WebSocket streaming | LangGraph `create_react_agent`; 3 tool đọc analyzer live; token quota; regex bóc URL tự đính frame; 4 kênh WS |
| `05-road-setup-runtime.md` | Hot-reload config + MongoDB write + API endpoints | Chỉnh ROI/zone live không restart; violation_sync tick 3s; 30+ endpoint chi tiết; scale zone về 600×400 |
| `06-diem-yeu-va-huong-phat-trien.md` | Khác biệt so với chatbot thị trường + limitations + roadmap | 4 điểm khác biệt thật sự; 9 điểm yếu tự công khai; 5 hướng phát triển xếp theo độ dễ + độ ăn điểm |
| `07-stack-cong-nghe.md` | Full stack + biến môi trường | FastAPI, Beanie, YOLOv8, OpenVINO, ByteTrack, EasyOCR, Gemini, LangGraph, React 19, shadcn/ui, framer-motion |
| `08-test-seed-data.md` | Test coverage + seed data + limitations | 3 test hiện có (rất mỏng); seed TrafficStat 6 khung giờ + 9 violation; 4 test cần bổ sung |
