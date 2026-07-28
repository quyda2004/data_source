# Traffic — Test & Seed Data

## Test & Seed data — trạng thái thật

**Source:** https://github.com/quyda2004/check/tree/main/backend/tests · https://github.com/quyda2004/check/blob/main/backend/seed.py

**Test hiện tại (`backend/tests/`):**
- `test_api.py` — **trống** (placeholder).
- `test_system_metrics.py` — kiểm tra shape output của `get_system_metrics()` (dùng cho endpoint admin).
- `test_tokenllm_model.py` — kiểm tra primary key `TokenLLM.user_id`.

**Seed (`backend/seed.py`, dòng 1–256):**
- Seed `TrafficStat`: mật độ theo **6 khung giờ** × **2 tuyến** (`Traffic`, `Traffic1`), hệ số 1.0 và 0.75 cho tuyến chính/phụ. Dùng cho biểu đồ dashboard "mật độ theo giờ trong tuần".
- Seed `Violation`: **9 vi phạm mẫu** với biển số hard-code, phân bố các loại lỗi (`VUOT_DEN_DO`, `LOI_RE_TRAI`, `LOI_RE_PHAI`), `final_status = CHO_KET_LUAN`.
- **Không** seed User admin → phải register thủ công + set `role_id=0` trong Mongo.
- Ảnh trong seed là **placeholder JPEG** chứa text mô tả vi phạm (không có ảnh thật), giúp UI có gì đó hiển thị khi demo mà không cần chạy analyzer.

> **Nói thẳng:** phần test hiện **rất mỏng** — chỉ 2 test có nội dung, không cover
> logic then chốt (crossing detection, event_id, quota, tool call). Chấm điểm sẽ trừ
> ở đây. Cần bổ sung tối thiểu:
> - `test_violation_crossing.py` — feed sequence điểm giả lập, assert `event_id` tạo
>   đúng lúc + không lặp.
> - `test_chatbot_tools.py` — mock analyzer state, gọi 3 tool, assert output shape.
> - `test_quota.py` — trừ quota tới 0, câu tiếp theo phải bị chặn.
> - **LLM eval** (LLM-as-a-judge trên bộ 20–30 câu hỏi chuẩn) — chưa có, mở rộng dài
>   hạn.
