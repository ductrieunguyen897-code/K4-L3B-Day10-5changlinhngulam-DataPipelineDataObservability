# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Họ và tên       | Vũ Hải Minh |
| MSSV               | 2A202602452 |
| Khóa/Lớp         | K4 - L3B |
| Tên nhóm         | 5changlinhngulam |
| Vai trò chính    | Corruption & integration owner |
| Repository         | https://github.com/ductrieunguyen897-code/K4-L3B-Day10-5changlinhngulam-DataPipelineDataObservability |
| Ngày hoàn thành | 2026-09-26 |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao  | Trạng thái                                 |
| ------------------ | --------------------- | ---------------- | ----------------- | -------------------------------------------- |
| Corruption suite | `src/ingestion/corruption.py` (`corrupt_clean_dataframe`) | Cleaned DataFrame | `data/clean/papers_clean_corrupted.*`, `data/results/corruption_log.json` | Hoàn thành |
| Baseline orchestration | `src/pipelines/phase1.py` (`run_phase1_pipeline`) | `Settings` | Toàn bộ artifact baseline (raw→clean→index→eval→quality→report) | Hoàn thành |
| Corruption & repair orchestration | `src/pipelines/corruption_flow.py` (`run_corruption_flow_pipeline`, `repair_from_raw_snapshot`) | Baseline artifacts + `Settings` | `data/results/corrupted_metrics.json`, `repaired_metrics.json`, `data/reports/corruption_report.md` | Hoàn thành |

Tôi chịu trách nhiệm điều phối tích hợp toàn bộ pipeline (chạy `script/run_phase1.py` và `script/run_corruption_flow.py` end-to-end), không tự sửa logic bên trong `crossref.py`/`cleaning.py`/`quality.py`/`testset.py` — các module đó do Thành viên 1 và 2 triển khai, tôi chỉ gọi qua interface đã thống nhất.

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động                         | Thành viên/module được hỗ trợ | Kết quả                    |
| ------------------------------------ | ------------------------------------ | ---------------------------- |
| Báo lỗi timezone trong `build_clean_dataframe` khi tích hợp lần đầu | Thành viên 1 — `cleaning.py` | Phát hiện lỗi khi chạy `run_phase1_pipeline`, báo lại để Thành viên 1 sửa `run_date.replace(tzinfo=timezone.utc)` |
| Đối chiếu số Expectation pass/fail trong `corrupted_quality_report.json` | Thành viên 2 — `quality.py` | Xác nhận đúng Expectation nào fail (`ExpectColumnValuesToBeUnique`) trước khi viết vào comparison report |

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao       | Cách xác minh         |
| --------------------------- | ----------------------------- | ------------------------- | ----------------------- |
| Implement `corrupt_clean_dataframe` (6 dạng lỗi) | `src/ingestion/corruption.py` | Corrupted DataFrame 23 dòng, log chi tiết 6 loại lỗi theo `paper_id` | `data/results/corruption_log.json` |
| Implement `run_phase1_pipeline` | `src/pipelines/phase1.py` | Baseline metrics: hit_rate=1.0, token_f1=0.70, judge_acc=0.70 | `python script/run_phase1.py` chạy thành công |
| Implement `run_corruption_flow_pipeline` + `repair_from_raw_snapshot` | `src/pipelines/corruption_flow.py` | Bảng so sánh 3 trạng thái, repaired metrics = baseline metrics | `python script/run_corruption_flow.py` chạy thành công |

Output cụ thể: console log của `run_corruption_flow_pipeline` in trực tiếp bảng so sánh 3 cột (baseline/corrupted/repaired) cho cả 4 metric — đây là bằng chứng tích hợp cho thấy toàn bộ chuỗi module của 3 thành viên hoạt động đúng khi ghép lại.

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Cần mô phỏng các sự cố dữ liệu thực tế (dữ liệu bị mất, bị nhiễu, bị trùng, bị cũ) để chứng minh: (a) Quality/Freshness Gate phát hiện được các sự cố đó (không phải "silent failure" hoàn toàn), (b) sự cố thực sự ảnh hưởng đến chất lượng câu trả lời của RAG agent, và (c) hệ thống có thể phục hồi từ nguồn tin cậy mà không cần gọi lại API ngoài.

### Cách triển khai

`corrupt_clean_dataframe` áp 6 phép biến đổi tuần tự trên một bản sao đã sort theo `published` giảm dần: drop 20% dòng mới nhất đầu tiên (mô phỏng mất dữ liệu do lỗi crawl gần đây), sau đó trên tập còn lại mới blank/inject-noise/truncate/stale-date theo tỉ lệ ~15-20% mỗi loại (các nhóm chỉ số không chồng lấn hoàn toàn để log rõ ràng từng loại), cuối cùng duplicate một số dòng đầu để mô phỏng lỗi ETL ghi trùng. Sau tất cả biến đổi, `summary_chars`/`text_for_embedding` được rebuild lại từ các cột đã bị sửa, đảm bảo embedding "nhìn thấy" đúng dữ liệu đã hỏng (không phải embedding cũ chưa cập nhật) — đây là điểm mấu chốt để observability và RAG agent phản ứng đúng với corruption.

`run_phase1_pipeline` xâu chuỗi: ingest (fetch-hoặc-load raw) → clean → ghi CSV/JSON → build ChromaDB index → build-hoặc-load test set → evaluate → quality+freshness → markdown report, mỗi bước dùng path từ `Settings.paths` để đảm bảo module khác đọc đúng vị trí.

`run_corruption_flow_pipeline` bắt buộc kiểm tra `clean_json`/`baseline_metrics` đã tồn tại trước khi chạy (raise lỗi rõ ràng nếu chưa chạy `run_phase1.py`), sau đó: corrupt → build index riêng (`papers-corrupted` collection) → evaluate (dùng **cùng** `eval_testset` đã sinh ở phase 1) → quality/freshness check → `repair_from_raw_snapshot()` (đọc lại raw records gốc, chạy lại `build_clean_dataframe` từ đầu — không sửa trực tiếp corrupted DataFrame) → build index mới (`papers-repaired`) → evaluate lại → sinh comparison report.

### Input, output và contract

| Thành phần                   | Mô tả                                     |
| ------------------------------ | ------------------------------------------- |
| Input                          | `Settings` (đường dẫn artifact cố định từ `core/config.py`), baseline artifacts đã tồn tại (`clean_json`, `baseline_metrics`) |
| Output                         | `dict` chứa `baseline_metrics/corrupted_metrics/repaired_metrics/*_quality/*_freshness`; markdown report tại `data/reports/corruption_report.md` |
| Module phụ thuộc             | `ingestion.cleaning.build_clean_dataframe`, `ingestion.corruption.corrupt_clean_dataframe`, `observability.quality.*`, `evaluation.metrics.evaluate_pipeline`, `retrieval.index.LocalEmbeddingIndex` |
| Module sử dụng output        | `report/*.md` (nguồn số liệu chính cho báo cáo nhóm) |
| Điều kiện lỗi cần xử lý | Chưa chạy `run_phase1.py` trước → raise `RuntimeError` rõ ràng thay vì lỗi `FileNotFoundError` mơ hồ |

### Cách xác minh

```bash
python script/run_phase1.py
python script/run_corruption_flow.py
```

- **Kết quả mong đợi:** Console in bảng so sánh baseline/corrupted/repaired cho 4 metric; file `data/reports/corruption_report.md` được tạo.
- **Kết quả thực tế:** Đúng như mong đợi — `retrieval_hit_rate: baseline=1.0 corrupted=0.6 repaired=1.0`, `mean_token_f1: baseline=0.7 corrupted=0.3788 repaired=0.7`, `judge_accuracy: baseline=0.7 corrupted=0.3 repaired=0.7`, `mean_judge_score: baseline=3.8 corrupted=2.7 repaired=3.8`.
- **Artifact/log:** `data/results/corrupted_metrics.json`, `data/results/repaired_metrics.json`, `data/reports/corruption_report.md` (không chứa secret; `.env` chỉ có `LLM_PROVIDER=openai`/`LLM_MODEL=gpt-4o-mini`, không commit key thật vào repo/report).

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Cần quyết định `repair_from_raw_snapshot()` nên "sửa" trực tiếp corrupted DataFrame (ví dụ: điền lại summary rỗng, xóa duplicate) hay build lại hoàn toàn từ raw.
- **Các phương án đã cân nhắc:** (1) Sửa tại chỗ (patch) corrupted DataFrame: bỏ duplicate bằng `drop_duplicates`, fill summary rỗng bằng giá trị mặc định; (2) Bỏ hoàn toàn corrupted DataFrame, đọc lại `crossref_records.json` (raw, chưa từng bị corrupt) và chạy lại `build_clean_dataframe` từ đầu.
- **Phương án đã chọn:** Phương án (2) — rebuild hoàn toàn từ raw snapshot.
- **Lý do:** Patch tại chỗ chỉ "che" triệu chứng (ví dụ xóa duplicate nhưng không đảm bảo hàng còn lại đúng nội dung gốc, hoặc fill summary bằng placeholder không phải nội dung thật) — không phản ánh đúng khái niệm "repair" trong data observability, vốn yêu cầu phục hồi *sự thật gốc* (raw preservation là mục đích chính của README bước 1). Rebuild từ raw đảm bảo repaired dataset có cùng provenance với baseline, nên nếu metrics của repaired khớp baseline, đó là bằng chứng mạnh rằng repair thực chất, không phải trùng hợp.
- **Bằng chứng quyết định phù hợp:** `repaired_metrics.json` khớp **chính xác từng số thập phân** với `baseline_metrics.json` (1.0/0.7/0.7/3.8) — nếu chỉ patch tại chỗ, khó có thể đạt độ khớp tuyệt đối này vì một số nội dung gốc (ví dụ title đầy đủ trước khi bị truncate) không thể suy ngược lại từ dữ liệu đã hỏng.

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:** Lần chạy tích hợp đầu tiên, `run_corruption_flow_pipeline` bị dừng giữa chừng khi tôi vô tình kill tiến trình đang chạy pipeline baseline để đổi cấu hình LLM provider, khiến `data/results/` và `data/reports/` trống dù `data/clean/papers_clean.csv` đã được ghi.
- **Lệnh hoặc bước tái hiện:** Chạy `python script/run_phase1.py`, kill tiến trình giữa lúc đang gọi LLM-judge (`evaluate_pipeline`), sau đó kiểm tra `ls data/results/ data/reports/` → rỗng.
- **Nguyên nhân gốc:** Pipeline không có cơ chế transactional/resume; nếu bị ngắt giữa chừng, các bước ghi file sau (metrics, report) đơn giản là chưa chạy tới, để lại trạng thái artifact không đầy đủ (chỉ có `clean/` nhưng thiếu `results/`/`reports/`).
- **Cách xử lý:** Chạy lại toàn bộ `python script/run_phase1.py` từ đầu sau khi cấu hình `.env` đã ổn định (`LLM_PROVIDER=openai`), không cố "chạy tiếp" từ trạng thái dở dang.
- **Cách xác minh sau khi sửa:** Chạy lại đầy đủ, xác nhận `data/results/baseline_metrics.json` và `data/reports/phase1_report.md` được tạo với timestamp mới, sau đó mới chạy `run_corruption_flow.py`.
- **Điều học được:** Khi đổi cấu hình môi trường (LLM provider) giữa lúc pipeline đang chạy, phải đảm bảo dừng và chạy lại toàn bộ từ đầu — không giả định các artifact dở dang là "hoàn chỉnh một phần" có thể tái sử dụng.

## 7. Hiểu biết về luồng end-to-end

**Câu trả lời:**

1. Dữ liệu đi từ Crossref → raw JSON → `PaperRecord` → cleaned DataFrame (Thành viên 1) → tôi dùng `LocalEmbeddingIndex.build()` để encode bằng MiniLM và nạp vào ChromaDB persistent client, tạo ra 3 collection riêng biệt (`papers-baseline`, `papers-corrupted`, `papers-repaired`) để 3 trạng thái không ghi đè lẫn nhau.
2. Evaluation set (Thành viên 2 sinh ra) được tôi tái sử dụng nguyên vẹn cho cả 3 lần gọi `evaluate_pipeline()` — không sinh lại — để `ground_truth_doc_ids` luôn nhất quán, cho phép so sánh `retrieval_hit_rate` một cách công bằng giữa 3 trạng thái.
3. Quality checks (GX Expectations, cấu trúc dữ liệu tại một thời điểm) khác Freshness monitoring (đo độ "cũ" theo `age_days`/thời gian) — trong pipeline của tôi, cả hai được gọi cùng lúc trong `run_data_quality_checks`/`build_freshness_report` nhưng ghi ra 2 file JSON riêng, để dễ truy vết corruption nào gây fail quality vs corruption nào gây fail freshness.
4. Dùng cùng test set cho 3 trạng thái là điều kiện bắt buộc để `run_corruption_flow_pipeline` không bị lỗi logic: nếu sinh lại test set trên corrupted data, một số `ground_truth_doc_ids` có thể trỏ đến tài liệu đã bị drop, khiến retrieval_hit_rate sai lệch không phản ánh đúng tác động thực của corruption.
5. Repair được xem là thành công dựa trên: artifact `repaired_clean_csv`/`repaired_clean_json` có đúng 24 dòng (bằng baseline), `repaired_quality_report.json.success=True`, `repaired_freshness_report.json.is_fresh=True`, và quan trọng nhất — `repaired_metrics.json` khớp chính xác `baseline_metrics.json` trên cả 4 chỉ số.

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal          | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| ---------------------- | -------: | --------: | -------: | ------------------------- |
| `retrieval_hit_rate` |   1.0000 |    0.6000 |   1.0000 | Đúng bằng baseline sau repair — repair không chỉ "gần đúng" mà khớp tuyệt đối |
| `mean_token_f1`      |   0.7000 |    0.3788 |   0.7000 | Khớp baseline tuyệt đối |
| `judge_accuracy`     |   0.7000 |    0.3000 |   0.7000 | Khớp baseline tuyệt đối |
| `mean_judge_score`   |   3.8000 |    2.7000 |   3.8000 | Khớp baseline tuyệt đối |
| Quality checks         |     Pass |      Fail |     Pass | Fail duy nhất do `paper_id` trùng lặp |
| Freshness status       |    Fresh |     Stale |    Fresh | stale_ratio quay về đúng 4.17% |

### Kết luận từ số liệu

1. Data corruption (tổng hợp cả 6 dạng lỗi, đặc biệt drop-latest làm mất tài liệu ground-truth khỏi index và duplicate làm vi phạm uniqueness) → quality gate signal chuyển Pass→Fail, freshness Fresh→Stale → agent metrics đồng loạt giảm (hit_rate -40%, token_f1 -45.9% tương đối, judge_accuracy -57.1% tương đối, judge_score -28.9% tương đối).
2. Repair action (rebuild toàn bộ từ raw snapshot, độc lập với corrupted data) → quality/freshness signal phục hồi hoàn toàn về đúng trạng thái baseline → agent metrics phục hồi **100%** về đúng giá trị baseline, không chỉ "cải thiện một phần" — đây là bằng chứng mạnh cho thấy pipeline repair đúng đắn về mặt kỹ thuật (không có data leakage hay ghi đè sai).

Corruption ảnh hưởng rõ nhất, từ góc độ orchestration/tích hợp: **drop latest records** là corruption nguy hiểm nhất vì nó xảy ra *trước khi build index*, nghĩa là tài liệu ground-truth hoàn toàn biến mất khỏi corpus retrieval được — không có cách nào agent "đoán đúng" dù prompt engineering có tốt đến đâu. Ngược lại, các corruption khác (noise, blank, truncate) chỉ làm giảm *chất lượng* câu trả lời trên tài liệu vẫn còn trong index.

Kết quả khác kỳ vọng: tôi ban đầu lo ngại `mean_judge_score` sẽ không phục hồi tuyệt đối về 3.8 vì LLM-judge (`gpt-4o-mini`) có thể cho điểm không hoàn toàn deterministic giữa các lần gọi (temperature=0 nhưng vẫn có thể có variance nhỏ). Thực tế kết quả khớp tuyệt đối 3.8000, cho thấy với `temperature=0.0` và câu hỏi/ngữ cảnh giống hệt baseline, judge trả về kết quả ổn định — điều này giúp tăng độ tin cậy khi dùng LLM-judge làm tiêu chí so sánh tự động giữa các lần chạy.

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. Orchestration pipeline cần validate tiền điều kiện rõ ràng (ví dụ: raise lỗi có thông điệp cụ thể nếu chưa chạy baseline) thay vì để lỗi runtime mơ hồ (`FileNotFoundError`) khi thiếu artifact phụ thuộc.
2. Corruption injection cần được thiết kế sao cho nó "chạm" vào đúng những Expectation/Freshness signal mà hệ thống quan sát đang theo dõi — nếu không, corruption sẽ trở thành "silent failure" thật sự (không bị observability bắt được, chỉ bị agent metric bắt được, hoặc tệ hơn là không bị gì bắt được).
3. Repair đáng tin cậy nhất khi nó độc lập hoàn toàn với dữ liệu đã hỏng (rebuild từ nguồn gốc), không phải sửa chữa tại chỗ — nguyên tắc "raw preservation" ở bước ingestion chính là nền tảng cho phép điều này khả thi.

### Nếu có thêm thời gian

Tôi sẽ thêm một bước "smoke test" tự động sau `run_phase1_pipeline()` kiểm tra `retrieval_hit_rate >= 0.8` trước khi cho phép `run_corruption_flow_pipeline()` chạy tiếp — tránh trường hợp baseline yếu (do lỗi cấu hình LLM/embedding) khiến toàn bộ so sánh 3 trạng thái mất ý nghĩa. Đo cải thiện bằng cách cố tình gây lỗi cấu hình (VD sai `embedding_model`) và xác nhận pipeline dừng sớm với thông báo rõ ràng thay vì chạy hết rồi mới phát hiện qua báo cáo.

## 10. Cam kết của thành viên

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi "đã chạy thành công" cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Vũ Hải Minh
**Ngày xác nhận:** 2026-09-26
