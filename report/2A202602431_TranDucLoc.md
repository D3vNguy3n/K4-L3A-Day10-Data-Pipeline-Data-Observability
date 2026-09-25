# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin | Nội dung |
| --- | --- |
| Họ và tên | Trần Đức Lộc |
| MSSV | `2A202602431` |
| Khóa/Lớp | K4 — L3A |
| Nhóm | `1Prompt4All` — K4-L3-DAY10 |
| Vai trò chính | RAG & Vector Index |
| Repository | [GitHub repository](https://github.com/D3vNguy3n/K4-L3A-Day10-Data-Pipeline-Data-Observability) |
| Ngày hoàn thành kỹ thuật | 2026-09-25 |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| --- | --- | --- | --- | --- |
| Embedding backend | `src/retrieval/embeddings.py`, `MiniLMEmbeddings` | `text_for_embedding` từ clean dataframe | Vector `sentence-transformers/all-MiniLM-L6-v2` chuẩn hóa | Hoàn thành |
| Vector store & indexing | `src/retrieval/index.py`, `LocalEmbeddingIndex.build/load/search/lookup` | Clean/corrupted/repaired dataframe và `Settings` | Ba Chroma collections `papers-baseline/corrupted/repaired` + manifest `data/embeddings/*.json` | Hoàn thành |
| QA retrieval | `src/retrieval/qa.py`, `answer_question()` | Câu hỏi benchmark, `Settings`, `LocalEmbeddingIndex` | `AnswerResult` với `retrieved_doc_ids/contexts/titles` và câu trả lời trích xuất | Hoàn thành |
| Agent tooling & LLM routing | `src/retrieval/agent.py`, `src/retrieval/llm.py` | `Settings` và index hiện tại | Agent với tools `semantic_search_papers`/`lookup_paper`, `build_llm` đa provider (gemini/openai/anthropic/openrouter/ollama/custom/mock) | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| --- | --- | --- |
| Thống nhất clean contract | Data Foundation (`cleaning.py`) | Xác nhận `text_for_embedding` ghép 5 trường Title/Authors/Published/Categories/Summary và `paper_id` unique trước khi index |
| Cô lập 3 trạng thái index | Pipeline Integrator (`phase1.py`, `corruption_flow.py`) | Mỗi trạng thái build collection riêng, không ghi đè lẫn nhau, cùng `top_k=4` và cùng embedding model |
| Đối chiếu metrics với retrieval | Observability & Evaluation | Hit rate suy giảm/phục hồi được giải thích bằng việc benchmark neo vào 5 docs mới nhất bị drop ở trạng thái corrupted |

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| --- | --- | --- | --- |
| Triển khai embedding backend | `src/retrieval/embeddings.py`, `MiniLMEmbeddings.embed_documents/embed_query` | Encode chuẩn hóa cosine, cache model qua `@lru_cache` | `python -c "import sentence_transformers; print('ok')"` và build index thành công |
| Build và persist 3 collections | `src/retrieval/index.py`, `LocalEmbeddingIndex.build()` | `data/chroma/` chứa 3 collections, `data/embeddings/papers_embeddings*.json` ghi manifest `backend/chroma`, `collection_name`, `documents` | `python script/run_phase1.py` và `python script/run_corruption_flow.py` exit 0; kiểm tra `collection.get()` đủ 24 docs/collection |
| Semantic search & exact lookup | `LocalEmbeddingIndex.search()`, `lookup()`, `semantic_search()` alias | Retrieval `top_k=4`, khoảng cách cosine chuyển thành `score = 1 - distance`, lookup theo `paper_id`/`title` lowercased | `baseline_answers.json` cho thấy mỗi câu hỏi trả về 4 contexts và `retrieval_hit=true` ở baseline/repaired |
| QA extraction | `src/retrieval/qa.py` | Trích `authors_joined`/`published`/`categories_joined` hoặc `first_sentence(summary)` tùy loại câu hỏi | `mean_token_f1=1.0` ở baseline/repaired, `0.007` ở corrupted |
| Đa provider LLM + mock fallback | `src/retrieval/llm.py`, `build_llm()` | Hỗ trợ `LLM_PROVIDER=mock` cho môi trường không có credential, các provider khác yêu cầu key tương ứng | `core/config.py:normalized_provider/require_llm_credentials` và `evaluation/metrics.py` fallback heuristic khi LLM judge không khả dụng |

Output tiêu biểu do phần việc này tạo ra là ba manifests `data/embeddings/papers_embeddings*.json` và ba Chroma collections; chúng là nguồn duy nhất cho `evaluation/metrics.py:evaluate_pipeline` tính `retrieval_hit_rate` và `mean_token_f1` trong `data/results/*.json`. Trạng thái hiện tại: baseline `hit 1.0/F1 1.0`, corrupted `hit 0.0/F1 0.007`, repaired `hit 1.0/F1 1.0`.

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Pipeline cần biến `text_for_embedding` đã chuẩn hóa thành vector có thể truy vấn ngữ nghĩa, đồng thời bảo đảm ba trạng thái baseline/corrupted/repaired được đánh giá công bằng trên cùng benchmark mà không làm nhiễm chéo dữ liệu. Retrieval phải vừa hỗ trợ tìm gần đúng (semantic search) vừa hỗ trợ truy vấn chính xác theo DOI/title để QA có thể neo đáp án.

### Cách triển khai

`MiniLMEmbeddings` wrap `SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")` với `normalize_embeddings=True`; `embed_documents` và `embed_query` đều trả về vector đã chuẩn hóa để Chroma `space: cosine` hoạt động đúng.

`LocalEmbeddingIndex.build(df, settings, embeddings_output_path)` thực hiện: `_build_documents` tạo `record_id = paper_id::index` và metadata đầy đủ; xác định `collection_name` qua `_derive_collection_name` (map 3 đường dẫn manifest chuẩn, còn lại slug theo stem); tạo/clear Chroma `PersistentClient(path=chroma_dir)`; `embed_documents` trên toàn bộ `content`; `collection.add(ids, embeddings, documents, metadatas)`; ghi manifest JSON gồm `backend`, `embedding_model`, `persist_path` (relative), `collection_name`, `documents`.

`LocalEmbeddingIndex.load()` đọc manifest và khôi phục client/collection. `search(query, top_k)` embed query, gọi `collection.query` với `n_results = min(top_k or settings.top_k, len(documents))`, chuyển `distance` thành `score`, trả về `SearchResult(paper_id, title, score, content, metadata)`. `lookup(value)` tra cứu O(1) qua hai map `documents_by_paper_id/title` đã lowercased.

`answer_question()` ưu tiên exact match: nếu câu hỏi chứa `'Title'` thì `lookup` trước, sau đó `search` và dedup (đưa exact lên đầu, cắt theo `top_k`). `_extract_answer` rẽ nhánh theo từ khóa câu hỏi để trả về `authors_joined`/`published`/`categories_joined` hoặc `first_sentence(summary)`.

`build_llm` và `build_agent` cung cấp routing đa provider và hai tools `semantic_search_papers`/`lookup_paper` cho agent; `RUN_RAGAS` và LLM judge được giữ optional để pipeline chạy offline với `LLM_PROVIDER=mock`.

### Input, output và contract

| Thành phần | Mô tả |
| --- | --- |
| Input | Clean dataframe có `paper_id, title, summary, authors_joined, categories_joined, published, text_for_embedding`; `Settings` chứa `embedding_model, chroma_dir, top_k, collection names` |
| Output | Ba manifests `data/embeddings/*.json`, ba Chroma collections mỗi collection 24 documents, `SearchResult` list và `AnswerResult` cho evaluation |
| Module phụ thuộc | `core/config`, `core/utils`, `ingestion/cleaning` (nguồn `text_for_embedding`), `evaluation/testset` (benchmark dùng để đo hit rate) |
| Module sử dụng output | `evaluation/metrics.py:evaluate_pipeline`, `pipelines/phase1.py`, `pipelines/corruption_flow.py`, `observability/reporting.py` |
| Điều kiện lỗi cần xử lý | Dataframe rỗng -> `ValueError`; collection chưa build -> `RuntimeError`; thiếu `paper_id/title/summary` -> cleaning đã loại trước khi index |

### Cách xác minh

```powershell
uv sync --python 3.12 --extra dev
uv run python script/run_phase1.py
uv run python script/run_corruption_flow.py
```

- **Kết quả mong đợi:** Hai lệnh exit 0; mỗi collection có 24 documents; baseline/repaired `retrieval_hit_rate=1.0`, corrupted `0.0` vì 5 ground-truth docs mới nhất bị drop.
- **Kết quả thực tế:** Đúng như mong đợi trong artifacts hiện tại: `baseline_metrics.json` và `repaired_metrics.json` đều `1.0/1.0`, `corrupted_metrics.json` `0.0/0.007`; `data/embeddings/*.json` tồn tại và `data/chroma/chroma.sqlite3` được tạo.
- **Artifact/log:** `data/embeddings/papers_embeddings*.json`, `data/chroma/`, `data/results/baseline_answers.json` (mỗi answer có 4 `retrieved_contexts` định dạng `Title/Authors/Published/Categories/Summary`).

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Nếu corruption và repair ghi đè cùng một Chroma collection thì không thể so sánh baseline/corrupted/repaired trên cùng benchmark mà không bị nhiễm chéo; đồng thời việc chạy lại repair sẽ không chứng minh được tính idempotent.
- **Các phương án đã cân nhắc:** (1) Dùng một collection duy nhất và build lại tuần tự; (2) Dùng ba collections độc lập `papers-baseline`, `papers-corrupted`, `papers-repaired` chung embedding model và `top_k`.
- **Phương án đã chọn:** Ba collections độc lập, mỗi trạng thái có manifest riêng và `persist_path` chung `data/chroma/`.
- **Lý do:** Cô lập trạng thái giúp đo silent failure chính xác, tránh contamination giữa clean và noisy vectors, dễ audit qua `collection_name` trong manifest, và giữ phép so sánh cùng cấu hình (cùng model, cùng benchmark, cùng `top_k`).
- **Bằng chứng quyết định phù hợp:** Ba collections đều tồn tại với 24 documents mỗi collection; `corrupted_metrics` giảm mạnh trong khi `repaired_metrics` trở về đúng baseline; chạy lại `run_corruption_flow.py` cho cùng repaired artifact (idempotent).

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:** `No module named 'numpy._core._multiarray_umath'` khi import `chromadb`/`sentence_transformers` bằng `.venv\Scripts\python.exe`; `python --version` báo `3.14`.
- **Lệnh hoặc bước tái hiện:** Kích hoạt `.venv` ban đầu và chạy `python -c "import chromadb, great_expectations, sentence_transformers; print('Môi trường sẵn sàng')"` hoặc `python script/run_phase1.py`.
- **Nguyên nhân gốc:** `.venv` được tạo bằng Python 3.14 trong khi các native wheels (`numpy`, `chromadb`, `sentence-transformers`) được build cho CPython 3.12 và `pyproject.toml` quy định `requires-python = ">=3.11,<3.14"`; ABI không đồng nhất nên extension không load được.
- **Cách xử lý:** Tái tạo môi trường bằng `uv sync --python 3.12 --extra dev` để interpreter và wheel ABI đồng nhất; không đổi code retrieval.
- **Cách xác minh sau khi sửa:** Import ba thư viện thành công và cả hai entrypoint `script/run_phase1.py` / `script/run_corruption_flow.py` chạy exit 0, sinh đủ manifests và metrics như trên.
- **Điều học được:** Với dependency có native extension, lockfile không thay thế việc chọn đúng Python runtime; phải cố định interpreter trong khoảng `pyproject.toml` cho phép trước khi cài.

## 7. Hiểu biết về luồng end-to-end

1. Crossref payload được parse thành `PaperRecord` và bảo toàn thành `crossref_records.json`. Cleaning chuẩn hóa text/date, tính `age_days`, deduplicate DOI và tạo `text_for_embedding` (Title/Authors/Published/Categories/Summary). Sau khi GX và freshness pass, `LocalEmbeddingIndex.build` embed toàn bộ documents bằng MiniLM và persist vào Chroma.
2. Mỗi câu benchmark có `ground_truth` và `ground_truth_doc_ids`. Hit rate đo retrieval có lấy đúng document chứa đáp án trong `top_k` hay không; token F1 đo mức trùng token giữa `answer` trích xuất và `ground_truth`; judge score bổ sung đánh giá correctness (hiện dùng heuristic fallback khi không có LLM credential).
3. Quality checks kiểm tra contract nội tại (row count, not null, uniqueness, summary length) trên dataframe hiện tại; freshness monitoring kiểm tra tính thời gian (`age_days > 180` vượt 25% thì `is_fresh=false`). Dữ liệu có thể pass quality nhưng vẫn fail freshness nếu quá cũ.
4. Ba trạng thái phải dùng cùng `data/eval/test_set.json` (10 câu, `benchmark_version=2`) để biến độc lập duy nhất là chất lượng dữ liệu/index. Đổi câu hỏi sẽ làm delta metrics không còn chứng minh tác động của corruption/repair.
5. Repair thành công khi: repaired clean artifact được tái tạo từ `crossref_records.json` (không dùng corrupted artifact), GX và freshness cùng pass, collection `papers-repaired` đủ 24 documents, và `repaired_metrics` trở về đúng `baseline_metrics` trên cùng benchmark; chạy lại repair cho cùng checksum.

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| --- | ---: | ---: | ---: | --- |
| `retrieval_hit_rate` | 1.000 | 0.000 | 1.000 | Mất 5 ground-truth docs mới nhất làm retrieval thất bại hoàn toàn - đây là tác động trực tiếp của vector store |
| `mean_token_f1` | 1.000 | 0.007 | 1.000 | Câu trả lời trên noisy context gần như không trùng đáp án |
| `judge_accuracy` | 1.000 | 0.000 | 1.000 | Correctness giảm theo retrieval quality |
| `mean_judge_score` | 5.000 | 1.000 | 5.000 | Corrupted chỉ đạt mức tối thiểu của heuristic |
| Quality checks | PASSED | FAILED | PASSED | Corrupted vi phạm uniqueness và summary length (mỗi cái 10/24) |
| Freshness status | PASSED | FAILED | PASSED | Stale ratio `4.2% -> 100% -> 4.2%`, phản ánh `stale_date -5y` trên 19 docs |

### Kết luận từ số liệu

1. Data corruption (drop 5 newest + blank/truncate/noise/duplicate/stale) -> GX fail, freshness fail (24/24 stale) -> `LocalEmbeddingIndex` trên corrupted collection trả về sai docs -> hit rate `1.0 -> 0.0` và token F1 `1.0 -> 0.007`.
2. Repair action (đọc lại `crossref_records.json`, chạy lại `build_clean_dataframe` và `LocalEmbeddingIndex.build` ra `papers-repaired`) -> quality/freshness phục hồi -> hit rate và token F1 trở về `1.0` trên cùng benchmark.

Corruption ảnh hưởng rõ nhất tới retrieval là `drop_latest_records` vì benchmark được `testset.py` neo có chủ đích vào 5 tài liệu mới nhất; khi các docs này vắng mặt trong corrupted index, hit rate chắc chắn về 0% bất kể ANN ranking. Các lỗi còn lại (noise prefix `ZXQ_CORRUPTED_VECTOR_NOISE`, blank summary, duplicate) tiếp tục làm giảm chất lượng vector và kích hoạt quality gate.

Điểm khác kỳ vọng ban đầu là ANN có thể phân giải các noisy near-ties khác nhau giữa những lần chạy, làm corrupted token F1 dao động rất nhỏ quanh `0.007`. Nhóm đã neo benchmark vào các document bị drop nên hit rate vẫn ổn định ở `0.0%`, và kết luận suy giảm/phục hồi không thay đổi.

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. Pipeline đáng tin cậy cần tách biệt embedding/index theo trạng thái; cùng một `text_for_embedding` với cùng model nhưng khác collection cho phép đo tác động của data quality lên retrieval một cách cô lập và tái lập.
2. Data quality và freshness bổ sung cho nhau: vector có thể đúng schema nhưng vẫn stale; cả hai tín hiệu đều cần pass trước khi serving.
3. Agent vẫn có thể trả lời trôi chảy khi retrieval sai (silent failure), nên chất lượng RAG phải được đo bằng hit rate/token F1/judge thay vì cảm nhận từ demo.

### Nếu có thêm thời gian

Bổ sung đánh giá embedding drift giữa baseline và corrupted (ví dụ cosine distance trung bình trên cùng query set), thử `top_k` khác nhau để vẽ precision@k, và bật `RUN_RAGAS=1` với credential hợp lệ để có thêm `faithfulness/context_precision/recall` bên cạnh hit rate hiện tại.

## 10. Cam kết của thành viên

Trần Đức Lộc xác nhận:

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi "đã chạy thành công" cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Trần Đức Lộc
**Ngày xác nhận:** 2026-09-25
