# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Họ và tên       | Đinh Lệnh Tiến Anh |
| MSSV               | 2A202602928 |
| Khóa/Lớp         | K4 - L3B|
| Tên nhóm         | 5changlinhngulam |
| Vai trò chính    | Data ingestion & cleaning owner |
| Repository         | https://github.com/ductrieunguyen897-code/K4-L3B-Day10-5changlinhngulam-DataPipelineDataObservability |
| Ngày hoàn thành | 2026-09-26 |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao  | Trạng thái                                 |
| ------------------ | --------------------- | ---------------- | ----------------- | -------------------------------------------- |
| Raw ingestion | `src/ingestion/crossref.py` (`parse_crossref_payload`, `fetch_source_records`, `load_raw_records`) | Crossref API query/filter từ `Settings` | `data/raw/crossref_response.json`, `data/raw/crossref_records.json` | Hoàn thành |
| Cleaning & data modeling | `src/ingestion/cleaning.py` (`build_clean_dataframe`) | List `PaperRecord` thô | `data/clean/papers_clean.csv`, `papers_clean.json`, cột `text_for_embedding`/`age_days` | Hoàn thành |

Tôi không nhận ownership cho quality gate, evaluation set hay corruption/repair — các phần này được thành viên 2 và 3 phụ trách trực tiếp.

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động                         | Thành viên/module được hỗ trợ | Kết quả                    |
| ------------------------------------ | ------------------------------------ | ---------------------------- |
| Kiểm tra dữ liệu repair (đối chiếu `papers_clean_repaired.csv` với `papers_clean.csv` gốc) | Thành viên 3 — `corruption_flow.py` | Xác nhận repaired dataset khớp 24/24 dòng, không còn duplicate, đúng schema clean |

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao       | Cách xác minh         |
| --------------------------- | ----------------------------- | ------------------------- | ----------------------- |
| Implement `parse_crossref_payload` | `src/ingestion/crossref.py` | Parse 24/24 item từ payload `message.items` thành `PaperRecord`, strip thẻ `<jats:p>` khỏi abstract | `python -c "from core.config import load_settings; from ingestion.crossref import fetch_source_records; ..."` → in `Đã tải 24 bài báo` |
| Implement `fetch_source_records` với retry + fallback offline | `src/ingestion/crossref.py` | Ghi raw response + raw records; tự retry 3 lần cho 429/503, fallback đọc snapshot nếu API lỗi | Kiểm tra `data/raw/crossref_response.json` tồn tại và khớp `message.items` gốc |
| Implement `build_clean_dataframe` | `src/ingestion/cleaning.py` | DataFrame 24 dòng sạch, có `age_days`, `text_for_embedding`, dedup theo `paper_id` | `python -c "... build_clean_dataframe(...)"` → in `Clean thành công 24 dòng` |

Output cụ thể: `data/clean/papers_clean.csv` có 24 dòng, mỗi dòng có `text_for_embedding` theo format `Title/Authors/Published/Categories/Summary` — output này được `retrieval/index.py` dùng trực tiếp để build ChromaDB collection.

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Dữ liệu Crossref trả về là JSON lồng nhiều cấp (`message.items[].author[]`, `.subject[]`, `.published.date-parts[][]`), abstract chứa thẻ JATS/XML rác (`<jats:p>...</jats:p>`). Cần chuẩn hóa thành schema phẳng, sạch, có định danh ổn định (`paper_id`) để các module phía sau (embedding, evaluation, quality) có thể dùng chung.

### Cách triển khai

`parse_crossref_payload` duyệt `payload["message"]["items"]`, dùng DOI làm `paper_id` (khóa tự nhiên, ổn định qua các lần fetch), loại bỏ thẻ HTML/XML bằng regex `<[^>]+>` rồi normalize khoảng trắng, bỏ record thiếu DOI/title. Ngày `published`/`created` được parse từ cấu trúc `date-parts` (year/month/day, mặc định month=1/day=1 nếu thiếu) sang chuỗi `YYYY-MM-DD`.

`build_clean_dataframe` tính `age_days = (run_date - published_date).days` theo UTC, ghép `authors_joined`/`categories_joined` bằng `compact_join`, tạo `text_for_embedding` theo mẫu 5 dòng cố định, sau đó `drop_duplicates(subset="paper_id", keep="first")` và sort theo `published` giảm dần để các bước sau (test set, corruption) luôn thao tác trên thứ tự nhất quán.

### Input, output và contract

| Thành phần                   | Mô tả                                     |
| ------------------------------ | ------------------------------------------- |
| Input                          | JSON payload Crossref (`message.items`), hoặc list `PaperRecord` đã parse |
| Output                         | `pandas.DataFrame` với cột `paper_id, title, summary, authors, authors_joined, categories, categories_joined, primary_category, published, updated, abs_url, pdf_url, age_days, summary_chars, text_for_embedding` |
| Module phụ thuộc             | `core.config.Settings`, `core.utils` (`normalize_whitespace`, `compact_join`, `write_json`) |
| Module sử dụng output        | `retrieval/index.py` (build embedding), `evaluation/testset.py` (sinh câu hỏi), `observability/quality.py` (chạy Expectations) |
| Điều kiện lỗi cần xử lý | Record thiếu DOI/title/summary hoặc `published` không parse được → bị loại khỏi DataFrame, không raise exception |

### Cách xác minh

```bash
python -c "from core.config import load_settings; from ingestion.crossref import fetch_source_records; s=load_settings(); r=fetch_source_records(s); print(f'Tín hiệu hoàn thành: Đã tải {len(r)} bài báo')"
python -c "from datetime import datetime, timezone; from core.config import load_settings; from ingestion.crossref import load_raw_records; from ingestion.cleaning import build_clean_dataframe; s=load_settings(); df=build_clean_dataframe(load_raw_records(s.paths.raw_records_json), datetime.now(timezone.utc)); print(f'Tín hiệu hoàn thành: Clean thành công {len(df)} dòng')"
```

- **Kết quả mong đợi:** `Đã tải 24 bài báo` và `Clean thành công 24 dòng`.
- **Kết quả thực tế:** Đúng như mong đợi, cả hai lệnh in đúng số 24.
- **Artifact/log:** `data/raw/crossref_response.json`, `data/raw/crossref_records.json`, `data/clean/papers_clean.csv` (không chứa secret).

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Cần chọn định danh duy nhất (`paper_id`) cho mỗi tài liệu, dùng xuyên suốt raw → clean → embedding → evaluation → corruption/repair.
- **Các phương án đã cân nhắc:** (1) Dùng index số thứ tự trong danh sách API trả về; (2) Dùng DOI (`item["DOI"]`) làm khóa.
- **Phương án đã chọn:** Dùng DOI làm `paper_id`.
- **Lý do:** Index số thứ tự không ổn định giữa các lần fetch (thứ tự trả về của Crossref có thể đổi, và sau khi drop/duplicate trong corruption thì index không còn ý nghĩa). DOI là khóa tự nhiên, duy nhất toàn cầu, không đổi qua các lần chạy — cần thiết để `ExpectColumnValuesToBeUnique(paper_id)` phát hiện đúng lỗi duplicate do corruption injection, và để `repair_from_raw_snapshot()` map lại đúng record gốc.
- **Bằng chứng quyết định phù hợp:** Sau corruption (duplicate rows), quality gate chính xác báo `paper_id` không unique (`corrupted_quality_report.json` → `success: False`); sau repair, `paper_id` unique trở lại 24/24 — chứng tỏ khóa DOI hoạt động đúng vai trò phát hiện lỗi.

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:** Ban đầu `build_clean_dataframe` raise lỗi khi tính `age_days` do so sánh datetime có timezone (`published_dt` là UTC-aware từ `pd.to_datetime(..., utc=True)`) với `run_date` không có timezone.
- **Lệnh hoặc bước tái hiện:** Gọi `build_clean_dataframe(records, datetime.now())` (không truyền UTC-aware datetime).
- **Nguyên nhân gốc:** Trộn lẫn datetime naive và aware khi trừ hai giá trị ngày tháng (`TypeError: can't subtract offset-naive and offset-aware datetimes`).
- **Cách xử lý:** Chuẩn hóa `run_date.replace(tzinfo=timezone.utc)` trước khi trừ, đảm bảo cả hai vế đều là UTC-aware.
- **Cách xác minh sau khi sửa:** Chạy lại lệnh kiểm tra bước 3 trong README, kết quả in đúng `Clean thành công 24 dòng` không lỗi.
- **Điều học được:** Luôn chuẩn hóa timezone tường minh khi làm việc với `pandas.to_datetime(utc=True)` kết hợp `datetime.now()` để tránh lỗi ngầm khó phát hiện.

## 7. Hiểu biết về luồng end-to-end

**Câu trả lời:**

1. Dữ liệu đi từ Crossref (JSON `message.items`) → `parse_crossref_payload` chuẩn hóa thành `PaperRecord` → `build_clean_dataframe` làm sạch, tính `age_days`, ghép `text_for_embedding` → `LocalEmbeddingIndex.build()` encode bằng MiniLM và nạp vào ChromaDB collection (`papers-baseline`/`papers-corrupted`/`papers-repaired`).
2. Evaluation set (10 câu, 4 loại) được sinh từ chính cleaned DataFrame, mỗi câu gắn `ground_truth_doc_ids=[paper_id]`. Khi đánh giá, `retrieval_hit_rate` kiểm tra xem `paper_id` retrieve được có nằm trong `ground_truth_doc_ids` không; `mean_token_f1`/LLM-judge so sánh câu trả lời agent sinh ra với `ground_truth`.
3. Quality checks (Great Expectations) kiểm tra tính đúng đắn tại một thời điểm (row count, null, uniqueness, độ dài summary) — là kiểm tra "point-in-time". Freshness monitoring đo lường theo thời gian (`age_days` so với ngưỡng 180 ngày, tỉ lệ stale so với 25%) — là kiểm tra "temporal decay", phản ánh dữ liệu có đang "cũ đi" hay không, độc lập với việc dữ liệu có hợp lệ về cấu trúc hay không.
4. Phải dùng cùng test set cho cả 3 trạng thái vì nếu đổi ground-truth, chênh lệch metric giữa các trạng thái sẽ lẫn cả nguyên nhân "đổi câu hỏi" lẫn "corruption/repair", làm mất khả năng quy kết nguyên nhân-kết quả.
5. Repair được coi là thành công khi: (a) `repaired_quality_report.json` có `success=True` và `row_count=24` giống baseline; (b) `repaired_freshness_report.json` có `is_fresh=True`, `stale_ratio` bằng đúng baseline (4.17%); (c) `repaired_metrics.json` có 4 chỉ số agent bằng đúng `baseline_metrics.json`.

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal          | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| ---------------------- | -------: | --------: | -------: | ------------------------- |
| `retrieval_hit_rate` |   1.0000 |    0.6000 |   1.0000 | Giảm mạnh vì drop-latest xóa mất một số tài liệu ground-truth khỏi index |
| `mean_token_f1`      |   0.7000 |    0.3788 |   0.7000 | Bị kéo xuống bởi blank summary + inject noise |
| `judge_accuracy`     |   0.7000 |    0.3000 |   0.7000 | LLM-judge phạt nặng câu trả lời rỗng/lẫn noise |
| `mean_judge_score`   |   3.8000 |    2.7000 |   3.8000 | Giảm ~1.1 điểm tuyệt đối |
| Quality checks         |     Pass |      Fail |     Pass | Fail do `paper_id` trùng lặp (duplicate injection) |
| Freshness status       |    Fresh |     Stale |    Fresh | Stale ratio vượt ngưỡng 25% do stale-date injection |

### Kết luận từ số liệu

1. Data corruption (drop latest 20% + duplicate rows + stale date +365 ngày) → `paper_id` mất duy nhất và `stale_ratio` tăng từ 4.17% lên 34.78% → quality/freshness signal chuyển Pass→Fail/Fresh→Stale → `retrieval_hit_rate` giảm từ 1.0 xuống 0.6 vì tài liệu ground-truth không còn trong index.
2. Repair action (rebuild từ `crossref_records.json` gốc, độc lập với dữ liệu đã corrupt) → quality/freshness signal phục hồi hoàn toàn (Pass/Fresh, stale_ratio 4.17%) → agent metric phục hồi đúng 100% về giá trị baseline.

Corruption ảnh hưởng rõ nhất là **drop latest records kết hợp duplicate rows**, vì đây là loại lỗi duy nhất trực tiếp làm thay đổi *tập tài liệu có trong index* (không chỉ làm bẩn nội dung), nên tác động trực tiếp lên `retrieval_hit_rate` — chỉ số nhạy nhất với việc thiếu tài liệu.

Kết quả khác kỳ vọng: tôi ban đầu dự đoán `truncate title` sẽ ảnh hưởng retrieval nhiều nhất (vì agent dùng regex tìm tiêu đề trong dấu nháy đơn để exact-match), nhưng vì retrieval vector-search (`index.search`) vẫn hoạt động dựa trên `text_for_embedding` đầy đủ (không chỉ title), nên tác động của truncate title chủ yếu rơi vào các câu hỏi dùng exact lookup theo title, không kéo tụt điểm tổng thể nhiều bằng drop-latest/duplicate.

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. Cần một khóa định danh (`paper_id`/DOI) ổn định xuyên suốt pipeline để mọi bước sau (quality check, corruption, repair) có thể tham chiếu và đối chiếu chính xác.
2. Data quality không chỉ là "dữ liệu có đúng định dạng" mà còn phải xét "dữ liệu có còn đúng theo thời gian" (freshness) — hai khía cạnh độc lập cần theo dõi riêng.
3. Corruption injection ảnh hưởng đến RAG agent không chỉ qua answer quality mà còn qua retrieval — khi tài liệu ground-truth biến mất khỏi index, agent không có cách nào "đoán đúng" dù LLM có mạnh đến đâu.

### Nếu có thêm thời gian

Tôi sẽ thêm cơ chế versioning cho raw snapshot (lưu timestamp fetch vào tên file), để có thể phục hồi về đúng phiên bản raw gần nhất khi có nhiều lần fetch, thay vì luôn ghi đè `crossref_response.json`. Đo cải thiện bằng cách kiểm tra `repair_from_raw_snapshot()` vẫn cho đúng kết quả khi có ≥2 lần fetch trước đó.

## 10. Cam kết của thành viên

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi "đã chạy thành công" cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** [Đinh Lệnh Tiến Anh]
**Ngày xác nhận:** 2026-09-26
