# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin | Nội dung |
| --- | --- |
| Họ và tên | Đặng Hữu Cương |
| MSSV | `2A202602572` |
| Khóa/Lớp | K4 — L3A |
| Nhóm | `1Prompt4All` — K4-L3-DAY10 |
| Vai trò chính | Observability & Evaluation |
| Repository | [GitHub repository](https://github.com/D3vNguy3n/K4-L3A-Day10-Data-Pipeline-Data-Observability) |
| Ngày hoàn thành | 2026-09-25 |

---

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| --- | --- | --- | --- | --- |
| Data Quality Gate (GX 1.x) | `src/observability/quality.py`<br>`run_data_quality_checks()` | Cleaned DataFrame, `Settings` | `baseline_quality_report.json`, `corrupted_quality_report.json`, `repaired_quality_report.json` | Hoàn thành |
| Freshness Monitoring SLA | `src/observability/quality.py`<br>`build_freshness_report()` | DataFrame chứa `published` & `age_days` | `freshness_report.json`, `corrupted_freshness_report.json`, `repaired_freshness_report.json` | Hoàn thành |
| Benchmark Test Set Builder | `src/evaluation/testset.py`<br>`build_test_set()`, `load_or_create_test_set()` | Cleaned DataFrame (`papers_clean.csv`) | `data/eval/test_set.json` (10 câu cố định) | Hoàn thành |
| Pipeline Evaluation & Metrics | `src/evaluation/metrics.py`<br>`evaluate_pipeline()`, `_token_f1()`, `_judge_answer()` | Index, test set JSON, `Settings` | `*_metrics.json` và `*_answers.json` (3 trạng thái) | Hoàn thành |
| Markdown Observability Reports | `src/observability/reporting.py`<br>`generate_phase1_report()`, `generate_corruption_report()` | Metrics JSON, Quality JSON, Freshness JSON | `data/reports/phase1_report.md`, `data/reports/corruption_report.md` | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| --- | --- | --- |
| Phản hồi schema cho Ingestion | Giang Thế Vũ (`cleaning.py`) | Đảm bảo cleaned DataFrame có đủ các cột `paper_id`, `title`, `summary`, `published`, `age_days`, `text_for_embedding` phục vụ đúng 6 Expectation của GX 1.x. |
| Đồng bộ contract Retrieval | Trần Đức Lộc (`retrieval/`) | Thống nhất cấu trúc `SearchResult` và trường `ground_truth_doc_ids` trong test set để tính chính xác `retrieval_hit_rate`. |
| Xác thực Idempotent Repair | Nguyễn Hoàng Lê Nguyên (`pipelines/`) | Cung cấp hàm validate và so sánh số liệu giữa baseline và repaired data để chứng minh pipeline phục hồi hoàn toàn. |

---

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| --- | --- | --- | --- |
| Xây dựng Quality Gate GX 1.x | `src/observability/quality.py` | 6 Expectations tự động kiểm định: row count, nullability, uniqueness, summary length | Chạy `run_data_quality_checks()`; baseline pass 6/6 (100%), corrupted fail 2/6 (66.7%) |
| Xây dựng Freshness SLA | `src/observability/quality.py` | Giám sát độ tươi với ngưỡng 180 ngày và tỷ lệ stale tối đa 25% | Baseline stale ratio = 4.2% (Pass); Corrupted stale ratio = 100.0% (Fail) |
| Thiết kế Benchmark 10 câu | `src/evaluation/testset.py` | `data/eval/test_set.json` gồm 10 câu bao phủ 4 dạng: `summary`, `authors`, `date`, `categories` | Đọc `data/eval/test_set.json`, kiểm tra `len(test_set) == 10` và versioning |
| Đánh giá định lượng 3 trạng thái | `src/evaluation/metrics.py` | Sinh 3 file metrics và 3 file answers chi tiết | Hit rate/F1: Baseline (`1.000/1.000`), Corrupted (`0.000/0.007`), Repaired (`1.000/1.000`) |
| Tự động hóa báo cáo Markdown | `src/observability/reporting.py` | `data/reports/phase1_report.md` và `data/reports/corruption_report.md` | Báo cáo Markdown được render tự động với bảng đối chiếu 3 cột trực quan |

### Output tiêu biểu

Output tiêu biểu là file [`data/reports/corruption_report.md`](file:///c:/Users/vietn/OneDrive/Documents/K4-L3A-Day10-Data-Pipeline-Data-Observability/data/reports/corruption_report.md). Báo cáo này tích hợp toàn bộ chỉ số từ Quality Gate, Freshness SLA và RAG Benchmark thành một bảng so sánh 3 trạng thái rõ ràng:
- **Baseline:** Hit rate 100%, F1 1.000, Quality Gate PASSED, Freshness SLA PASSED.
- **Corrupted:** Hit rate 0.0%, F1 0.007, Quality Gate FAILED (vi phạm summary length và unique `paper_id`), Freshness SLA FAILED (stale ratio 100%).
- **Repaired:** Hit rate 100%, F1 1.000, Quality Gate PASSED, Freshness SLA PASSED.

---

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Trong các hệ thống RAG thực tế, dữ liệu bẩn và lỗi thời thường gây ra hiện tượng **Silent Failure**: AI Agent vẫn phản hồi tự tin, trôi chảy nhưng ngữ cảnh bị sai lệch, dẫn tới hallucination mà không có bất kỳ Exception nào được ném ra. Vai trò Observability & Evaluation cần thiết lập:
1. **Chốt kiểm dịch dữ liệu tự động (Quality Gate & Freshness SLA)** để phát hiện và cảnh báo dữ liệu xấu/cũ trước khi nạp vào vector store.
2. **Bộ Benchmark và Metric đa tầng** để đo lường định lượng mức độ suy giảm chất lượng câu trả lời khi dữ liệu bị lỗi và chứng minh sự phục hồi hoàn toàn sau khi sửa chữa.

### Cách triển khai

1. **Great Expectations 1.x Fluent Pandas API:**
   - Khởi tạo Ephemeral Context trên bộ nhớ RAM: `gx.get_context(mode="ephemeral")`.
   - Định nghĩa Pandas Data Source và DataFrame Asset:
     ```python
     data_source = context.data_sources.add_pandas(name="papers_source")
     data_asset = data_source.add_dataframe_asset(name="papers_asset")
     batch_definition = data_asset.add_batch_definition_whole_dataframe("papers_batch")
     batch = batch_definition.get_batch(batch_parameters={"dataframe": df})
     ```
   - Áp dụng 6 Expectations:
     - `ExpectTableRowCountToBeBetween(min_value=5, max_value=5000)`
     - `ExpectColumnValuesToNotBeNull` cho `paper_id`, `title`, `text_for_embedding`
     - `ExpectColumnValuesToBeUnique(column="paper_id")`
     - `ExpectColumnValueLengthsToBeBetween(column="summary", min_value=30)`
2. **Freshness SLA Monitoring:**
   - Tính toán `stale_rows` dựa trên `age_days > freshness_threshold_days` (180 ngày).
   - Đánh giá `is_fresh = (stale_ratio <= 0.25)`.
3. **Deterministic Test Set Builder:**
   - Lấy 5 bài báo mới nhất và sinh 10 câu hỏi (2 câu/bài) qua 4 loại nghiệp vụ: `summary`, `authors`, `date`, `categories`.
   - Chiến lược neo vào 5 bài báo mới nhất giúp test set cực kỳ nhạy cảm với lỗi "Drop latest records" (vốn loại bỏ 20% bản ghi mới nhất), phản ánh tức thì sự sụp đổ của retrieval.
4. **Đo lường Retrieval & Generation Metrics:**
   - `retrieval_hit_rate`: Tỷ lệ câu hỏi mà `retrieved_doc_ids` chứa ít nhất một `ground_truth_doc_ids`.
   - `token_f1`: Điểm F1 trùng khớp token giữa câu trả lời dự đoán và đáp án chuẩn.
   - LLM Judge: Phân tích tính đúng đắn với thang điểm 1–5 (có heuristic fallback khi offline).

### Input, output và contract

| Thành phần | Mô tả |
| --- | --- |
| Input | `pd.DataFrame` sau làm sạch hoặc tiêm lỗi, cấu hình `Settings` từ `src/core/config.py` |
| Output | `data/eval/test_set.json`, `data/quality/*.json`, `data/results/*.json`, `data/reports/*.md` |
| Module phụ thuộc | `src/ingestion/cleaning.py` (cung cấp DataFrame), `src/retrieval/index.py` (cung cấp index) |
| Module sử dụng output | `src/pipelines/phase1.py` và `src/pipelines/corruption_flow.py` (dùng quality checks làm điều kiện dừng / gating) |
| Điều kiện lỗi cần xử lý | DataFrame thiếu cột bắt buộc, số lượng bài báo < 10 không đủ sinh test set, LLM provider timeout khi judge |

### Cách xác minh

```powershell
# Chạy kiểm tra Quality Gate độc lập
python -c "from core.config import load_settings; from observability.quality import run_data_quality_checks; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); res=run_data_quality_checks(df, s, 'test'); print(f'Quality status: {res[\"success\"]}')"

# Chạy kiểm tra Test Set Builder độc lập
python -c "from core.config import load_settings; from evaluation.testset import build_test_set; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); ts=build_test_set(df, s.paths.eval_testset); print(f'Sinh thanh cong {len(ts)} cau hoi')"

# Chạy toàn bộ pipeline Phase 1 và Phase 2
python script/run_phase1.py
python script/run_corruption_flow.py
```

- **Kết quả mong đợi:** Baseline Quality Gate pass (100%), 10 câu hỏi test được tạo ra, baseline hit rate đạt 1.0, corrupted hit rate giảm sâu, repaired hit rate phục hồi 1.0.
- **Kết quả thực tế:** Tất cả các lệnh chạy thành công, exit code 0; số liệu khớp chính xác từng file JSON artifacts.
- **Artifact/log:** `data/quality/baseline_quality_report.json`, `data/results/baseline_metrics.json`, `data/reports/corruption_report.md`.

---

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Lựa chọn phương pháp thiết kế bộ Benchmark Test Set giữa hai phương án: (A) Sinh ngẫu nhiên câu hỏi bằng LLM mỗi lần chạy, hoặc (B) Thiết kế bộ 10 câu hỏi mẫu cố định (Deterministic Rule-based Benchmark) bám chặt vào các trường dữ liệu thực tế và phiên bản hóa (`benchmark_version=2`).
- **Các phương án đã cân nhắc:**
  - *Phương án A:* Dùng LLM sinh động câu hỏi. Ưu điểm là đa dạng ngữ nghĩa, nhưng nhược điểm là không ổn định giữa các lần chạy (non-deterministic), tốn chi phí token/quota, và khó cô lập được biến số thực nghiệm khi so sánh 3 trạng thái dữ liệu.
  - *Phương án B:* Dùng template định hướng theo 4 nghiệp vụ học thuật (`summary`, `authors`, `date`, `categories`) neo vào 5 bài báo đầu tiên của danh sách đã sort.
- **Phương án đã chọn:** Phương án B kết hợp cơ chế caching thông minh qua `load_or_create_test_set()`.
- **Lý do:** Để đánh giá chính xác tác động của Data Quality và Data Corruption, bộ câu hỏi đánh giá phải là **bất biến (invariant)** xuyên suốt cả 3 trạng thái (Baseline, Corrupted, Repaired). Nếu bộ câu hỏi thay đổi sau mỗi lần chạy, sự thay đổi điểm số sẽ bị nhiễu do câu hỏi khác nhau chứ không phản ánh đúng chất lượng dữ liệu.
- **Bằng chứng quyết định phù hợp:** Nhờ cố định câu hỏi dựa trên 5 bài báo mới nhất, khi luồng corruption tiêm lỗi "Drop latest 20% records" (bỏ 5/24 bài báo mới nhất), retrieval hit rate sụt giảm chính xác và ổn định về `0.0%`, tạo ra bằng chứng không thể chối cãi về hiện tượng Silent Failure.

---

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:**
  ```text
  AttributeError: 'DataContext' object has no attribute 'sources'
  # Hoặc TypeError khi gọi context.sources.add_pandas(...)
  ```
- **Lệnh tái hiện:**
  ```python
  import great_expectations as gx
  context = gx.get_context()
  context.sources.add_pandas("papers_source")
  ```
- **Nguyên nhân gốc:** Mã nguồn ban đầu tham khảo các tutorial cũ của Great Expectations phiên bản 0.17/0.18 (dùng cú pháp `context.sources`). Trong Great Expectations 1.x, kiến trúc Core Fluent API đã thay đổi hoàn toàn: `sources` bị thay thế bằng `data_sources`, và việc validate một DataFrame in-memory yêu cầu chuỗi cấu hình: `context.data_sources.add_pandas() -> add_dataframe_asset() -> add_batch_definition_whole_dataframe() -> get_batch()`.
- **Cách xử lý:** Viết lại toàn bộ hàm `run_data_quality_checks()` trong `src/observability/quality.py` theo đúng chuẩn GX 1.x Ephemeral Context:
  ```python
  context = gx.get_context(mode="ephemeral")
  data_source = context.data_sources.add_pandas(name="papers_source")
  data_asset = data_source.add_dataframe_asset(name="papers_asset")
  batch_definition = data_asset.add_batch_definition_whole_dataframe("papers_batch")
  batch = batch_definition.get_batch(batch_parameters={"dataframe": df})
  ```
- **Cách xác minh sau khi sửa:** Chạy kiểm thử kiểm định chất lượng, script thực thi trơn tru trong 0.2 giây, trả về đúng dictionary kết quả và ghi file `baseline_quality_report.json` với `gx_success: true`.
- **Điều học được:** Khi làm việc với các thư viện data observability phát triển nhanh như Great Expectations, cần bám sát tài liệu chính thức của major release (1.x) thay vì sử dụng snippet cũ từ các bài hướng dẫn chưa cập nhật.

---

## 7. Hiểu biết về luồng end-to-end

1. **Chu trình dữ liệu:** Dữ liệu thô từ Crossref API được lưu trữ nguyên vẹn để phục vụ Data Lineage. Sau đó, module cleaning chuẩn hóa text, tính `age_days`, tạo `text_for_embedding` và khử trùng lặp. Trước khi vector hóa, Data Quality Gate (GX 1.x) và Freshness SLA kiểm tra tính toàn vẹn. Nếu đạt, mô hình `all-MiniLM-L6-v2` nhúng text thành vector và nạp vào ChromaDB.
2. **Cơ chế đánh giá:** Bộ benchmark 10 câu hỏi cố định truy vấn vào hệ thống RAG. Hệ thống đo lường cả 2 chặng: chặng tìm kiếm (`retrieval_hit_rate` kiểm tra xem tài liệu đúng có lọt vào top-k hay không) và chặng sinh câu trả lời (`token_f1` và LLM Judge kiểm tra mức độ chính xác ngữ nghĩa của câu trả lời).
3. **Ý nghĩa của Quality Gate & Freshness SLA:** Quality checks kiểm soát tính toàn vẹn cấu trúc (không null, không trùng, đủ độ dài), còn Freshness SLA kiểm soát tính kịp thời của tri thức. Cả hai tạo thành tấm lá chắn kép ngăn chặn rác dữ liệu lọt vào serving layer.
4. **Tính chất bất biến của Benchmark:** Cần giữ nguyên một test set duy nhất cho cả 3 trạng thái để đảm bảo biến số độc lập duy nhất trong thực nghiệm là **chất lượng dữ liệu trong Vector Database**.
5. **Đặc trưng của Idempotent Repair:** Phục hồi an toàn là quá trình tái tạo lại dữ liệu sạch trực tiếp từ bản lưu trữ thô (`data/raw/crossref_records.json`) thay vì cố gắng chắp vá dữ liệu đã bị biến dạng. Dù chạy lại luồng repair bao nhiêu lần, kết quả dữ liệu sạch, vector embeddings và điểm số đánh giá luôn đồng nhất 100%.

---

## 8. Phân tích kết quả

### Metrics chính

| Metric / Tín hiệu | Baseline (Sạch) | Corrupted (Tiêm lỗi) | Repaired (Phục hồi) | Nhận xét chi tiết |
| --- | ---: | ---: | ---: | --- |
| `retrieval_hit_rate` | **1.000** (100%) | **0.000** (0%) | **1.000** (100%) | Khi mất 20% bài báo mới nhất, bộ tìm kiếm hoàn toàn không tìm thấy ground-truth docs |
| `mean_token_f1` | **1.000** | **0.007** | **1.000** | Điểm tương đồng token sụp đổ gần như bằng 0 trên dữ liệu lỗi |
| `judge_accuracy` | **1.000** (100%) | **0.000** (0%) | **1.000** (100%) | Đánh giá tính chính xác câu trả lời giảm về 0% do context sai lệch |
| `mean_judge_score` | **5.000** | **1.000** | **5.000** | Điểm chất lượng trung bình giảm từ mức tối đa (5/5) xuống tối thiểu (1/5) |
| Quality Checks (GX) | **PASSED** (6/6) | **FAILED** (4/6) | **PASSED** (6/6) | Bắt được 2 vi phạm: summary rỗng/ngắn và trùng lặp mã `paper_id` |
| Freshness SLA | **PASSED** (4.2%) | **FAILED** (100%) | **PASSED** (4.2%) | Tỷ lệ bài báo cũ vượt ngưỡng cảnh báo đỏ khi bị lùi ngày |

### Kết luận từ số liệu

1. **Minh chứng Silent Failure:** Khi dữ liệu bị tiêm lỗi, Agent không hề báo lỗi crash chương trình mà vẫn trả lời dựa trên các context sai/nhiễu được nhúng trong ChromaDB. Nếu không có Quality Gate cảnh báo sớm, người dùng cuối sẽ nhận được thông tin sai lệch nghiêm trọng.
2. **Năng lực phát hiện của Observability:** Cả 2 tầng giám sát đều hoạt động hoàn hảo: GX 1.x bắt đứng lỗi schema/content, trong khi Freshness SLA phát hiện ngay hiện tượng dữ liệu bị "mốc meo".
3. **Hiệu quả của Idempotent Repair:** Cơ chế phục hồi dữ liệu từ bản thô ban đầu giúp toàn bộ các chỉ số kỹ thuật và điểm số chất lượng RAG quay trở về trạng thái hoàn hảo ban đầu (Hit Rate 1.0, Token F1 1.0).

---

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. **Tầm quan trọng của Data Observability:** Trong kiến trúc RAG, dữ liệu là nền móng. Một mô hình LLM tiên tiến nhất vẫn sẽ trả lời sai nếu Vector Database bị đầu độc bởi dữ liệu bẩn. Việc cài đặt Quality Gate chặn ngay trước bước Embed là yêu cầu sống còn.
2. **Làm chủ thư viện chuẩn thế hệ mới:** Việc chuyển đổi thành công sang Great Expectations 1.x giúp pipeline chạy nhanh hơn (nhờ ephemeral in-memory validation) và mã nguồn tinh gọn, hiện đại hơn.
3. **Thiết kế thực nghiệm có đối chứng:** Cần kiểm soát chặt chẽ các biến số (như cố định test set, tách biệt collections) thì kết quả đo lường sụt giảm và phục hồi mới có tính thuyết phục khoa học cao.

### Nếu có thêm thời gian

- Xây dựng thêm một **Observability Dashboard** trực quan bằng Streamlit để hiển thị trực tiếp biểu đồ phân bố `age_days` và trạng thái pass/fail của các Expectation theo thời gian thực (đạt điểm bonus B1).
- Mở rộng thêm các Expectation kiểm tra Semantic Drift (độ lệch phân bố vector embedding) giữa các đợt nạp dữ liệu.

---

## 10. Cam kết của thành viên

Đặng Hữu Cương xác nhận:

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Đặng Hữu Cương  
**Ngày xác nhận:** 2026-09-25
