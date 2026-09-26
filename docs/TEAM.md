# Danh Sách Thành Viên & Báo Cáo Phân Công Nhóm

- **Tên Nhóm:** `5changlinhngulam`
- **Mã Nhóm / Lớp:** `K4 - Lớp B (Ca Sáng)`
- **Tên Repository Nộp Bài:** `K4-L3B-Day10-5changlinhngulam-DataPipelineDataObservability`

---

## # Thành viên

| STT | Họ và tên | MSSV | Email | Vai trò & Phân công công việc | Báo cáo cá nhân |
|---:|---|---|---|---|---|
| 1 | Đinh Lệnh Tiến Anh | 2A202602928 | dinhlenhtienanh.forwork@gmail.com | Data ingestion & cleaning owner (`src/ingestion/crossref.py`, `src/ingestion/cleaning.py`) | `report/individual_2A202602928_DinhLenhTienAnh.md` |
| 2 | Nguyễn Đức Triệu | 2A202602978 | ductrieunguyen897@gmail.com | Evaluation & observability owner (`src/evaluation/testset.py`, `src/observability/quality.py`, `src/observability/reporting.py`) | `report/individual_2A202602978_NguyenDucTrieu.md` |
| 3 | Vũ Hải Minh | 2A202602452 | vhaiminh.2k5@gmail.com | Corruption & integration owner (`src/ingestion/corruption.py`, `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py`) | `report/individual_2A202602452_VuHaiMinh.md` |

---

## # Cá nhân

### ## Đinh Lệnh Tiến Anh - 2A202602928
- **Vai trò:** Data ingestion & cleaning owner.
- **Công việc chi tiết đã hoàn thành:**
  - Xây dựng module thu thập Crossref API (`api.crossref.org/works`) với retry/backoff mũ (2s/4s/8s) cho status 429/503 và fallback đọc snapshot offline khi API lỗi, trong `src/ingestion/crossref.py`.
  - Chuẩn hóa schema, strip thẻ JATS khỏi `summary`, dedup theo `paper_id`, tính `age_days` và ghép `text_for_embedding` trong `src/ingestion/cleaning.py`.
  - Đảm bảo raw records (`crossref_records.json`) được bảo toàn nguyên vẹn làm nguồn phục hồi cho bước repair.
- **Điều học được / Đóng góp chính:**
  - Kỹ thuật truy vết nguồn gốc dữ liệu (Data Lineage) và bảo toàn raw snapshot trước khi biến đổi để phục vụ repair đáng tin cậy.

### ## Nguyễn Đức Triệu - 2A202602978
- **Vai trò:** Evaluation & observability owner.
- **Công việc chi tiết đã hoàn thành:**
  - Xây dựng bộ evaluation set 10 câu hỏi (4 dạng: summary/authors/date/categories) trong `src/evaluation/testset.py`.
  - Thiết lập Data Quality Gate với Great Expectations 1.x (6 expectations: row count, not-null, uniqueness, length) và Freshness Monitoring (ngưỡng 25% stale > 180 ngày) trong `src/observability/quality.py`.
  - Xây dựng `src/observability/reporting.py` để sinh báo cáo so sánh baseline/corrupted/repaired.
- **Điều học được / Đóng góp chính:**
  - Cách thiết lập hệ thống cảnh báo sớm (quality gate + freshness SLA) để chặn đứng hiện tượng Silent Failure trước khi dữ liệu vào serving layer.

### ## Vũ Hải Minh - 2A202602452
- **Vai trò:** Corruption & integration owner.
- **Công việc chi tiết đã hoàn thành:**
  - Xây dựng module tiêm lỗi (`src/ingestion/corruption.py`) với 6 dạng: drop latest 20%, blank summary, inject noise, truncate title, stale date, duplicate rows.
  - Xâu chuỗi pipeline end-to-end trong `src/pipelines/phase1.py` (`run_phase1_pipeline`) và `src/pipelines/corruption_flow.py` (`run_corruption_flow_pipeline`), đảm bảo dùng chung một evaluation set cho baseline/corrupted/repaired.
  - Cài đặt `repair_from_raw_snapshot()` để rebuild dữ liệu từ raw snapshot gốc, đảm bảo repair có provenance độc lập với dữ liệu bị corrupt.
- **Điều học được / Đóng góp chính:**
  - Thiết kế Idempotent Pipeline và quản lý trạng thái luồng dữ liệu đa tầng (baseline → corrupted → repaired) để so sánh nhân quả chính xác.
