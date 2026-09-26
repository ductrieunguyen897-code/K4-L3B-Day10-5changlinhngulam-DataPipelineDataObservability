# Group Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin bài nộp

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Khóa/Lớp         | K4 - L3B    |
| Tên nhóm         | 5changlinhngulam            |
| Repository         | https://github.com/ductrieunguyen897-code/K4-L3B-Day10-5changlinhngulam-DataPipelineDataObservability |
| Ngày hoàn thành | 2026-09-26                  |

### Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò chính | Module/deliverable sở hữu |
| --: | --- | --- | --- | --- |
| 1 | Đinh Lệnh Tiến Anh | 2A202602928 | Data ingestion & cleaning owner | `src/ingestion/crossref.py`, `src/ingestion/cleaning.py` — raw records, cleaned dataset |
| 2 | Nguyễn Đức Triệu | 2A202602978 | Evaluation & observability owner | `src/evaluation/testset.py`, `src/observability/quality.py`, `src/observability/reporting.py` |
| 3 | Vũ Hải Minh | 2A202602452 | Corruption & integration owner | `src/ingestion/corruption.py`, `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py` |

## 2. Tóm tắt kết quả

Nhóm đã hoàn thiện toàn bộ khung pipeline: thu thập dữ liệu từ Crossref REST API (24 bài báo), làm sạch và chuẩn hóa dữ liệu (`age_days`, `text_for_embedding`), lập chỉ mục ChromaDB với embedding MiniLM, sinh bộ evaluation set 10 câu hỏi (4 dạng: summary/authors/date/categories), và triển khai Data Quality Gate với Great Expectations 1.x (Ephemeral Context) kết hợp Freshness Monitoring (ngưỡng 25% stale > 180 ngày).

Baseline pipeline (`script/run_phase1.py`) chạy thành công, tạo đầy đủ artifact: `papers_clean.csv` (24 dòng), `baseline_metrics.json` (`retrieval_hit_rate=1.0`, `mean_token_f1=0.70`, `judge_accuracy=0.70`, `mean_judge_score=3.8`), quality gate **PASS** (6/6 expectations), freshness **fresh** (stale_ratio 4.17%).

Corruption flow (`script/run_corruption_flow.py`) tiêm đủ 6 dạng lỗi (drop latest 20%, blank summary, inject noise, truncate title, stale date, duplicate rows). Corruption ảnh hưởng rõ nhất là **drop latest records + duplicate rows**, khiến `paper_id` mất tính duy nhất và stale_ratio tăng vọt lên 34.78% (>25% ngưỡng) → quality gate **FAIL**, kéo theo retrieval_hit_rate giảm còn 0.60 và mean_token_f1 giảm còn 0.379 (silent failure quan sát được qua agent trả lời sai/không đầy đủ). Sau khi `repair_from_raw_snapshot()` phục hồi từ `crossref_records.json` gốc, toàn bộ metric quay lại đúng bằng baseline (retrieval_hit_rate=1.0, mean_token_f1=0.70, judge_accuracy=0.70, quality gate PASS, is_fresh=True).

Giới hạn quan trọng nhất: Ragas evaluation bị skip (`RUN_RAGAS` chưa bật) do runtime lâu; test set chỉ có 10 câu trên 24 tài liệu nên một số corruption case (ví dụ blank summary) không luôn rơi trúng tài liệu nằm trong ground-truth doc id, khiến quan sát silent-failure phụ thuộc một phần vào ngẫu nhiên của sample corrupt.

## 3. Kiến trúc và luồng dữ liệu

### Luồng end-to-end

```text
Crossref API (api.crossref.org/works)
    -> raw response (data/raw/crossref_response.json) / raw records (data/raw/crossref_records.json)
    -> cleaning: age_days, text_for_embedding (data/clean/papers_clean.csv|json)
    -> embedding MiniLM (all-MiniLM-L6-v2) + ChromaDB index (data/embeddings/papers_embeddings.json)
    -> evaluation baseline (data/results/baseline_metrics.json)
    -> quality/freshness reports (data/quality/)
    -> corruption (6 scenarios, data/results/corruption_log.json)
    -> re-index và re-evaluate corrupted (data/results/corrupted_metrics.json)
    -> repair từ raw snapshot (data/clean/papers_clean_repaired.*)
    -> re-evaluate repaired (data/results/repaired_metrics.json)
    -> comparison report (data/reports/corruption_report.md)
```

### Trách nhiệm của từng khối

| Khối             | Input          | Xử lý chính             | Output/artifact          | Owner          |
| ----------------- | -------------- | -------------------------- | ------------------------ | -------------- |
| Ingestion         | Crossref API query "agentic retrieval augmented generation large language model" | Fetch với retry (429/503, backoff mũ), fallback đọc snapshot offline khi API lỗi | `data/raw/crossref_response.json`, `data/raw/crossref_records.json` | Thành viên 1 |
| Cleaning          | 24 `PaperRecord` thô | Normalize whitespace, tính `age_days`, ghép `text_for_embedding`, dedup theo `paper_id` | `data/clean/papers_clean.csv`, `papers_clean.json` | Thành viên 1 |
| Embedding/index   | Cleaned DataFrame | MiniLM (`all-MiniLM-L6-v2`) qua ChromaDB `PersistentClient`, cosine space | `data/embeddings/papers_embeddings.json` + `data/chroma/` | Thành viên 3 (orchestration), dùng module chung `retrieval/index.py` |
| Evaluation        | Cleaned DataFrame, 10 câu test | Sinh 4 loại câu hỏi (summary/authors/date/categories), đánh giá retrieval hit + token F1 + LLM-judge | `data/eval/test_set.json`, `data/results/baseline_metrics.json` | Thành viên 2 |
| Observability     | Cleaned/corrupted/repaired DataFrame | 6 Great Expectations (row count, not-null, uniqueness, length) + freshness SLA (ngưỡng 180 ngày / 25% stale) | `data/quality/*_quality_report.json`, `*_freshness_report.json` | Thành viên 2 |
| Corruption/repair | Cleaned DataFrame | Drop 20% mới nhất, blank/inject noise summary, truncate title, stale date +365 ngày, duplicate rows; repair = rebuild từ raw snapshot | `data/results/corruption_log.json`, `data/clean/papers_clean_corrupted.*`, `papers_clean_repaired.*` | Thành viên 3 |
| Orchestration     | Toàn bộ module trên | `run_phase1_pipeline()` và `run_corruption_flow_pipeline()` xâu chuỗi end-to-end, cùng dùng một evaluation set | `data/reports/phase1_report.md`, `data/reports/corruption_report.md` | Thành viên 3 |

## 4. Cách tái hiện kết quả

### Cấu hình không chứa secret

| Biến/cấu hình             | Giá trị sử dụng |
| ---------------------------- | ------------------- |
| `LLM_PROVIDER`             | `openai` |
| `LLM_MODEL`                | `gpt-4o-mini` |
| Embedding model              | `sentence-transformers/all-MiniLM-L6-v2` |
| Số lượng Crossref records | `max_results=24` (24 bài báo thu được) |
| Retrieval`top_k`           | `4` |
| Freshness threshold          | `180` ngày, ngưỡng stale-ratio cảnh báo `25%` |
| Random seed, nếu có        | Không dùng random seed cố định (test set chọn theo thứ tự `published` giảm dần) |

Không dán nội dung API key hoặc file `.env` vào báo cáo. `.env` chỉ chứa `LLM_PROVIDER`/`LLM_MODEL` và các key được nạp qua `python-dotenv`, không commit vào repo.

### Lệnh cài đặt

```bash
source .venv/bin/activate
python -m pip install -e .
```

### Lệnh chạy

Baseline:

```bash
python script/run_phase1.py
```

Corruption flow:

```bash
python script/run_corruption_flow.py
```

### Kết quả tái hiện

| Lệnh             | Trạng thái                                    | Thời điểm chạy gần nhất | Bằng chứng                         |
| ----------------- | ----------------------------------------------- | ----------------------------- | ------------------------------------ |
| Baseline pipeline | Thành công | 2026-09-26 (UTC 03:13-03:14) | `data/reports/phase1_report.md`, `data/results/baseline_metrics.json` |
| Corruption flow   | Thành công | 2026-09-26 (UTC 03:14-03:15) | `data/reports/corruption_report.md`, `data/results/corrupted_metrics.json`, `repaired_metrics.json` |

## 5. Ingestion, cleaning và data contract

### Nguồn dữ liệu

| Thuộc tính                | Giá trị                             |
| --------------------------- | ------------------------------------- |
| Source                      | Crossref REST API (`api.crossref.org/works`), fallback offline snapshot `data/raw/crossref_response.json` |
| Query/filter                | `query="agentic retrieval augmented generation large language model"`, `filter=from-pub-date:<today-180d>,has-abstract:true` |
| Thời điểm lấy dữ liệu | 2026-09-26 |
| Số record nhận được    | 24 |
| Cơ chế retry/backoff      | 3 lần thử, backoff mũ (2s/4s/8s) cho status 429/503; nếu vẫn lỗi thì đọc lại snapshot đã lưu trước đó |

### Raw và clean schema

| Trường        | Kiểu dữ liệu | Bắt buộc?  | Ý nghĩa   | Xử lý khi thiếu/sai |
| --------------- | --------------- | ------------ | ----------- | ---------------------- |
| `paper_id` (DOI) | str          | Có         | Khóa duy nhất định danh bài báo | Bỏ record nếu thiếu DOI |
| `title`         | str            | Có         | Tiêu đề    | Bỏ record nếu title rỗng |
| `summary`       | str            | Có         | Tóm tắt (đã loại thẻ JATS/HTML) | Bỏ record nếu summary rỗng sau khi strip tag |
| `authors`       | list[str]      | Không      | Danh sách tác giả | Giữ list rỗng nếu Crossref không trả `author` |
| `categories`    | list[str]      | Không      | Lĩnh vực chuyên môn (`subject`) | Giữ list rỗng, `primary_category=""` |
| `published`     | str (YYYY-MM-DD) | Có       | Ngày xuất bản, dùng tính `age_days` | Bỏ record nếu không parse được ngày |
| `age_days`      | int            | Có (sinh ra ở bước clean) | Tuổi dữ liệu = run_date - published | Tính lại mỗi lần chạy pipeline |

### Quy tắc cleaning

| Quy tắc                                 | Quality dimension liên quan | Số record bị tác động | Cách xác minh      |
| ---------------------------------------- | ---------------------------- | -------------------------: | -------------------- |
| Loại record thiếu `paper_id`/`title`/`summary` hoặc `published` không parse được | Completeness/Validity | 0/24 (không có record nào bị loại trong snapshot mẫu) | So sánh `len(raw_records)` với `len(clean_df)` |
| Dedup theo `paper_id`, giữ bản ghi đầu tiên | Uniqueness | 0/24 (dataset gốc không trùng) | `df["paper_id"].duplicated().sum() == 0` |
| Strip thẻ JATS (`<jats:p>...</jats:p>`) khỏi `summary` | Validity | 24/24 (tất cả abstract đều bọc trong `<jats:p>`) | Kiểm tra `summary` không còn ký tự `<` |

Cách tạo `text_for_embedding`, document ID và `age_days`:

`paper_id` = DOI trả về từ Crossref (đã là định danh duy nhất, ổn định qua các lần fetch). `age_days` = `(run_date - published_date).days`, tính lại theo thời điểm chạy pipeline hiện tại (UTC). `text_for_embedding` ghép 5 trường theo mẫu cố định: `Title / Authors / Published / Categories / Summary`, đảm bảo mọi baseline/corrupted/repaired dùng cùng một cấu trúc để so sánh embedding công bằng.

## 6. Evaluation setup

| Thành phần                             | Cấu hình thực tế          |
| ---------------------------------------- | ----------------------------- |
| Số câu hỏi                            | 10 |
| Các`question_type`                    | `summary`, `authors`, `date`, `categories` (xoay vòng đều trên 10 tài liệu đầu tiên theo `published` giảm dần) |
| Ground-truth document ID                 | `[paper_id]` của chính tài liệu được dùng để sinh câu hỏi |
| Embedding model                          | `sentence-transformers/all-MiniLM-L6-v2` |
| Vector store/collection                  | ChromaDB `PersistentClient` tại `data/chroma/`; collection `papers-baseline` / `papers-corrupted` / `papers-repaired` |
| Retrieval`top_k`                       | 4 |
| LLM provider/model                       | `openai` / `gpt-4o-mini` (dùng cho LLM-judge trong `evaluation/metrics.py`) |
| Test set dùng chung cho ba trạng thái | `data/eval/test_set.json` (sinh một lần từ baseline, tái sử dụng nguyên vẹn cho corrupted và repaired) |

Vì sao test set được giữ nguyên khi đánh giá baseline, corrupted và repaired: `run_corruption_flow_pipeline()` không gọi lại `build_test_set`, mà đọc lại `settings.paths.eval_testset` đã được `run_phase1_pipeline()` sinh ra. Nhờ vậy, cùng một tập câu hỏi/ground-truth được dùng cho cả 3 lần đánh giá, nên chênh lệch metric phản ánh đúng tác động của corruption/repair, không phải do đổi tập test.

## 7. Kết quả baseline

### Artifact checklist

| Artifact                 | Đường dẫn thực tế                | Trạng thái | Ghi chú   |
| ------------------------ | -------------------------------------- | ------------ | ---------- |
| Raw response/records     | `data/raw/crossref_response.json`, `crossref_records.json` | Có | 24 bài báo |
| Cleaned dataset          | `data/clean/papers_clean.csv`, `papers_clean.json` | Có | 24 dòng |
| Embedding manifest/index | `data/embeddings/papers_embeddings.json` | Có | backend `chroma`, model MiniLM |
| Evaluation set           | `data/eval/test_set.json` | Có | 10 câu, 4 loại |
| Baseline metrics         | `data/results/baseline_metrics.json` | Có | xem bảng dưới |
| Quality/freshness        | `data/quality/baseline_quality_report.json`, `freshness_report.json` | Có | PASS / fresh |
| Baseline report          | `data/reports/phase1_report.md`      | Có | sinh tự động |

### Baseline metrics

| Metric                 |       Giá trị | Diễn giải                             |
| ---------------------- | --------------: | --------------------------------------- |
| `retrieval_hit_rate` |     1.0000 | Cả 10/10 câu hỏi retrieval trúng đúng `paper_id` ground-truth trong top-4 |
| `mean_token_f1`      |     0.7000 | Câu trả lời trích xuất trực tiếp từ metadata (authors/date/categories) khớp cao; câu `summary` phụ thuộc câu đầu tiên nên F1 thấp hơn kéo trung bình xuống |
| `judge_accuracy`     |     0.7000 | 7/10 câu được LLM-judge (`gpt-4o-mini`) đánh giá đúng về mặt nội dung |
| `mean_judge_score`   |     3.8000 | Điểm trung bình 1-5, phản ánh chất lượng câu trả lời khá tốt nhưng chưa hoàn hảo |
| Ragas, nếu có        | Bỏ qua (`RUN_RAGAS` chưa bật) | Runtime lâu hơn đáng kể so với thời lượng bài lab, nhóm không bật để giữ pipeline nhanh |

## 8. Data quality và freshness

### Quality checks

| Check        | Quality dimension | Ngưỡng/kỳ vọng | Kết quả baseline      | Bằng chứng |
| ------------ | ----------------- | ------------------ | ----------------------- | ------------ |
| `ExpectTableRowCountToBeBetween` | Completeness | 5–5000 dòng | Pass (24 dòng) | `baseline_quality_report.json` |
| `ExpectColumnValuesToNotBeNull` (paper_id, title, text_for_embedding) | Completeness | Không null | Pass (3/3 cột) | `baseline_quality_report.json` |
| `ExpectColumnValuesToBeUnique` (paper_id) | Uniqueness | Không trùng | Pass | `baseline_quality_report.json` |
| `ExpectColumnValueLengthsToBeBetween` (summary) | Validity | ≥30 ký tự | Pass | `baseline_quality_report.json` |

### Freshness

| Thuộc tính               | Giá trị                           |
| -------------------------- | ----------------------------------- |
| Freshness được đo tại | `data/clean/papers_clean.csv` (baseline) |
| Timestamp mới nhất       | 2026-07-22 |
| Ngưỡng freshness         | `age_days > 180` được tính là stale; cảnh báo nếu tỉ lệ stale > 25% |
| Trạng thái baseline      | Fresh |
| Lý do                     | Chỉ 1/24 bài (4.17%) có `age_days > 180`, thấp hơn nhiều so với ngưỡng 25% → `is_fresh=True` |

## 9. Corruption scenarios và repair

| Corruption         | Cách tạo | Record bị tác động | Quality signal kỳ vọng | Tác động thực tế | Cách repair   |
| ------------------ | ---------- | ---------------------: | ------------------------ | --------------------- | -------------- |
| Drop latest records | Bỏ 20% bài mới nhất theo `published` | 4/24 | Giảm coverage, có thể mất tài liệu ground-truth | Một phần retrieval miss vì tài liệu ground-truth không còn trong index | Rebuild từ raw snapshot |
| Blank summary | Xóa trắng `summary` ở một số dòng | 3/20 (sau khi drop) | `ExpectColumnValueLengthsToBeBetween(summary)` fail | Trả lời `summary` rỗng/không có nội dung cho câu hỏi loại `summary` | Repair khôi phục `summary` gốc |
| Inject noise | Chèn chuỗi rác `###CORRUPTED_NOISE_$$$` vào `summary` | 3/20 | Giảm chất lượng ngữ nghĩa embedding | Câu trả lời lẫn ký tự rác, giảm token F1 | Repair loại bỏ noise |
| Truncate title | Cắt `title` còn 6 ký tự | 3/20 | Mất semantic trong tiêu đề dùng để lookup exact-match | Một số câu hỏi dạng "paper '<title>'" không lookup được chính xác | Repair khôi phục title đầy đủ |
| Stale date | Lùi `published` 365 ngày, `age_days += 365` | 4/20 | Tăng `stale_ratio`, có thể vi phạm freshness SLA | `stale_ratio` từ 4.17% → 34.78% (>25%) → `is_fresh=False` | Repair tính lại `age_days` từ raw + `run_date` hiện tại |
| Duplicate rows | Nhân đôi 3 dòng đầu | 3 dòng thêm (23 dòng tổng, do đã drop 4 dòng trước đó: 20+3) | `ExpectColumnValuesToBeUnique(paper_id)` fail | Quality gate **FAIL** vì `paper_id` trùng lặp | Repair rebuild từ raw records, không còn duplicate |

Corruption log:

- Đường dẫn: `data/results/corruption_log.json`
- Trạng thái: Có
- Nhận xét: Log ghi đầy đủ 6 khóa (`dropped_latest`, `blanked_summary`, `noise_injected`, `truncated_title`, `stale_date`, `duplicated`), mỗi khóa liệt kê danh sách `paper_id` bị tác động, đủ để truy vết record nào bị biến đổi bởi corruption nào.

Repair đảm bảo phục hồi từ nguồn đáng tin cậy: `repair_from_raw_snapshot()` **không** sửa lại dữ liệu đã bị corrupt (không "vá" summary rỗng hay xóa duplicate thủ công), mà đọc lại `data/raw/crossref_records.json` — bản ghi gốc chưa từng bị biến đổi — và chạy lại toàn bộ `build_clean_dataframe()` từ đầu. Vì vậy repaired dataset có provenance độc lập với corrupted dataset, không phải chỉ che giấu triệu chứng lỗi.

## 10. So sánh baseline, corrupted và repaired

| Metric/signal            | Baseline | Corrupted | Repaired | Thay đổi do corruption | Mức phục hồi | Nhận xét   |
| ------------------------ | -------: | --------: | -------: | -----------------------: | --------------: | ------------ |
| `retrieval_hit_rate`   |   1.0000 |    0.6000 |   1.0000 |                  -0.4000 |           100% | Repair phục hồi hoàn toàn hit rate |
| `mean_token_f1`        |   0.7000 |    0.3788 |   0.7000 |                  -0.3212 |           100% | Repair phục hồi hoàn toàn |
| `judge_accuracy`       |   0.7000 |    0.3000 |   0.7000 |                  -0.4000 |           100% | Repair phục hồi hoàn toàn |
| `mean_judge_score`     |   3.8000 |    2.7000 |   3.8000 |                  -1.1000 |           100% | Repair phục hồi hoàn toàn |
| Quality checks pass/fail |     Pass |      Fail |     Pass |          Pass→Fail       |     Phục hồi Pass | `paper_id` trùng lặp là nguyên nhân chính gây fail |
| Freshness status         |    Fresh |     Stale |    Fresh |    4.17%→34.78% stale     | Phục hồi 4.17% | Stale date injection đẩy vượt ngưỡng 25% |

Kết luận nhân quả:

1. Corruption (drop 20% mới nhất + duplicate rows + stale date +365 ngày) → `paper_id` mất tính duy nhất và `stale_ratio` vượt 25% (34.78%) → data quality gate chuyển từ Pass sang **Fail** → retrieval_hit_rate giảm từ 1.0 xuống 0.6 vì một số tài liệu ground-truth bị drop khỏi index và câu trả lời agent bị nhiễu bởi noise/blank summary.
2. Repair action (`repair_from_raw_snapshot()` rebuild từ `crossref_records.json` gốc) → quality gate quay lại **Pass**, freshness quay lại **Fresh** (stale_ratio 4.17%) → toàn bộ 4 agent metric (`retrieval_hit_rate`, `mean_token_f1`, `judge_accuracy`, `mean_judge_score`) phục hồi về đúng giá trị baseline, chứng minh dữ liệu được phục hồi thực chất từ nguồn tin cậy chứ không phải che giấu lỗi.

## 11. Vấn đề tích hợp quan trọng

- **Triệu chứng:** Khi mới chạy `run_corruption_flow_pipeline()`, nếu gọi `evaluate_pipeline` với một `test_set_path` khác baseline, kết quả so sánh baseline/corrupted/repaired sẽ lệch nhau vì khác ground-truth, làm sai lệch kết luận về tác động của corruption.
- **Nguyên nhân:** `run_phase1_pipeline()` sinh test set nếu chưa tồn tại; nếu corruption flow vô tình gọi lại `build_test_set()` trên dữ liệu đã corrupt, ground-truth doc id có thể trỏ đến các bài đã bị drop.
- **Cách xử lý:** `run_corruption_flow_pipeline()` chỉ đọc lại `settings.paths.eval_testset` đã có sẵn (không sinh mới), đảm bảo baseline, corrupted, repaired dùng chung một bộ 10 câu hỏi/ground-truth.
- **Cách xác minh:** So sánh `data/eval/test_set.json` (checksum/nội dung không đổi) trước và sau khi chạy `script/run_corruption_flow.py`; đối chiếu `ground_truth_doc_ids` trong `baseline_answers.json`, `corrupted_answers.json`, `repaired_answers.json` — cả 3 file đều có cùng `id`/`question`/`ground_truth_doc_ids`.

## 12. Giới hạn và hướng cải thiện

| Giới hạn hiện tại | Ảnh hưởng   | Hướng cải thiện có thể kiểm chứng |
| --------------------- | -------------- | ----------------------------------------- |
| Ragas metrics bị skip (`RUN_RAGAS` mặc định tắt) | Thiếu góc nhìn faithfulness/context precision-recall độc lập với LLM-judge tự viết | Bật `RUN_RAGAS=1` trong CI/nightly run riêng, không chặn vòng lặp lab chính |
| Test set chỉ 10 câu trên 24 tài liệu | Một số corruption (vd blank summary trên tài liệu không nằm trong test set) không luôn thể hiện rõ trên metric | Tăng `TARGET_QUESTION_COUNT` hoặc lấy mẫu ngẫu nhiên có seed cố định để bao phủ nhiều tài liệu hơn |
| Freshness threshold cố định 25%/180 ngày, hard-code trong `quality.py` | Không linh hoạt cho các domain dữ liệu có tốc độ cập nhật khác nhau | Đưa `FRESHNESS_STALE_RATIO_THRESHOLD` vào `Settings`/`.env` để cấu hình theo từng pipeline |

## 13. Checklist trước khi nộp

- [x] Thông tin nhóm và repository chính xác.
- [x] Phân công khớp với module, artifact và kết quả thực tế.
- [x] Lệnh tái hiện đã được chạy lại trên phiên bản dùng để nộp.
- [x] Baseline, corrupted và repaired dùng cùng evaluation set.
- [x] Bảng metrics khớp với các file trong `data/results/`.
- [x] Quality/freshness conclusions khớp với `data/quality/`.
- [x] Các đường dẫn báo cáo và artifact truy cập được.
- [x] Mỗi thành viên có một `individual_report.md` riêng về vai trò và phần việc của mình (điền tên/MSSV thật trước khi nộp).
- [x] Không có `.env`, API key hoặc secret trong repository, report hoặc log.
