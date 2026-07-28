# Traffic — Video Pipeline & Detection Models

## Kiến trúc sơ lược
Mỗi tuyến 1 process con (Python multiprocessing). Analyzer chạy trong process con, endpoint FastAPI chỉ đọc shared memory (`Manager().dict()`). Vòng lặp: frame → YOLOv8 → ByteTrack → tính speed → check violation → OCR queue → sync Mongo mỗi 3s. Video gốc scale về 600×400 khi phân tích.

## Tool / Library dùng
- **YOLOv8l** (Ultralytics) — 3 model train: vehicle 6-class, helmet binary, plate 1-class → [backend/app/ai_models](https://github.com/quyda2004/check/tree/main/backend/app/ai_models)
- **OpenVINO INT8** — export production, CPU 9-10 FPS → [backend/app/ai_models/model%20N/cvt_model.ipynb](https://github.com/quyda2004/check/blob/main/backend/app/ai_models/model%20N/cvt_model.ipynb)
- **ByteTrack** — qua Ultralytics `.track(tracker="bytetrack.yaml")`, confidence 0.15
- **Ultralytics `solutions.SpeedEstimator`** — monocular 2D positioning
- **EasyOCR + Gemini Vision fallback** — đọc biển số → [backend/app/modules/plate_ocr_service.py](https://github.com/quyda2004/check/blob/main/backend/app/modules/plate_ocr_service.py)
- **Roboflow hosted API** — helmet detection
- **Python multiprocessing** (`Process` + `Manager()`) — shared dict/list giữa parent + workers

## Source chính
- Analyzer multiprocessing: [backend/app/services/road_services/AnalyzeOnRoadForMultiProcessing.py](https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/AnalyzeOnRoadForMultiProcessing.py)
- Analyzer base (sequential, dùng cho test): [backend/app/services/road_services/AnalyzeOnRoadBase.py](https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/AnalyzeOnRoadBase.py)
- Violation detection: [backend/app/services/road_services/ViolationAnalyzer.py](https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/ViolationAnalyzer.py)
- Violation sync (Mongo tick 3s): [backend/app/services/road_services/violation_sync.py](https://github.com/quyda2004/check/blob/main/backend/app/services/road_services/violation_sync.py)
- Traffic light detector (pixel-based, không ML): [backend/app/modules/traffic_light/detector.py](https://github.com/quyda2004/check/blob/main/backend/app/modules/traffic_light/detector.py)
- Config path model: [backend/app/core/config.py](https://github.com/quyda2004/check/blob/main/backend/app/core/config.py)
- Báo cáo đồ án (paper reference, số liệu): [bao_cao_do_an.md](https://github.com/quyda2004/check/blob/main/bao_cao_do_an.md)
- Paper reference: [refer_paper](https://github.com/quyda2004/check/tree/main/refer_paper)

## Số liệu chốt
- Vehicle mAP@0.5: **0.909** · Helmet mAP@0.5: **91%** · Plate F1: **0.9968** · OCR char accuracy: **98%**
- End-to-end: **9-10 FPS** trên CPU · Latency frame→UI **<200ms** · RAM/process **500MB-1GB**
- Vi phạm hỗ trợ: `VUOT_DEN_DO`, `LOI_RE_TRAI`, `LOI_RE_PHAI`, `KHONG_MU_BAO_HIEM`
- Event ID format: `road:vehicle_id:frame:type` (idempotent upsert)
- Paper reference: Sadik et al. [1], Zhang et al. [2] ByteTrack, Lian et al. [3], Aquitan et al. [4], Moussaoui et al. [5]
