# Hướng dẫn làm bài Day 14 — AI Evaluation & Benchmarking Pipeline

Tài liệu này được viết sau khi rà soát toàn bộ mã nguồn, README, đề bài, worksheet, test suite, validator, golden dataset mẫu và 10 tài liệu corpus trong repo. Hãy làm **đúng thứ tự** dưới đây. Đặc biệt: không gọi OpenAI API trước khi golden dataset đã được validator xác nhận `PASS`.

## 1. Hiểu bài cần làm gì

Bài lab có ba khối tách biệt:

1. `domain_assistant.py` là **hệ thống RAG được đánh giá**. Nó dùng BM25 lấy các đoạn từ corpus, gửi question và retrieved chunks cho OpenAI, rồi lưu câu trả lời thật.
2. `template.py` là **evaluation engine** mà bạn phải hoàn thiện. Nó chấm answer-side metrics, retrieval-side metrics, chạy benchmark, regression và failure analysis.
3. `evaluate_answers.py` là **adapter**. Nó ghép `golden_dataset.json` với `artifacts/actual_answers.json`, gọi evaluation engine, in bảng và lưu kết quả.

Luồng hoàn chỉnh:

```text
Corpus → viết golden_dataset.json → validate PASS
       → domain_assistant.py → artifacts/actual_answers.json
       → evaluate_answers.py → artifacts/benchmark_results.json
       → điền exercises.md và reflection.md
       → copy template.py sang solution/solution.py → kiểm tra cuối
```

Bốn file bắt buộc phải nộp:

- `solution/solution.py`
- `golden_dataset.json`
- `exercises.md`
- `reflection.md`

Hai file trong `artifacts/` là kết quả hỗ trợ phân tích, không phải deliverable bắt buộc.

## 2. Trạng thái hiện tại của repo

Ở thời điểm tạo hướng dẫn này:

- `template.py` vẫn còn tất cả TODO/`NotImplementedError` bắt buộc.
- `golden_dataset.json` đã có đúng 20 slot nhưng question, expected answer và evidence còn trống.
- `solution/` chưa có `solution.py`.
- `exercises.md` và `reflection.md` vẫn là worksheet cần điền.
- `.env` có tồn tại và đã được `.gitignore` bỏ qua. Không đưa API key vào bài viết, source code hoặc commit.

Vì test ưu tiên `solution/solution.py` nếu file đó tồn tại, trong giai đoạn làm core nên sửa `template.py`. Chỉ copy sang `solution/solution.py` ở bước gần cuối; nếu sửa template sau đó thì phải copy lại.

## 3. Bước 1 — Chuẩn bị môi trường

Mở PowerShell tại thư mục gốc repo và chạy:

```powershell
Get-Location
py -0p
python --version
```

Python phải từ 3.11 trở lên. Repo đã có `.venv`; thử kích hoạt:

```powershell
.venv\Scripts\Activate.ps1
python --version
python -m pip install -r requirements.txt
python -c "import openai, dotenv, pytest; print('Environment OK')"
```

Nếu `.venv` dùng Python cũ hoặc hỏng, tạo lại bằng Python 3.11+ theo `guide_lab.md`. Dependencies của bài là `openai`, `python-dotenv` và `pytest`.

### Kiểm tra hoàn tất bước 1

- `python --version` là 3.11+.
- Lệnh kiểm tra import in `Environment OK`.

### Làm gì tiếp theo?

Chạy baseline tests ở bước 2. Chưa sửa code trước khi xác nhận test có thể collect bình thường.

## 4. Bước 2 — Chạy baseline và làm Part 1

Chạy:

```powershell
pytest tests/ -v
```

Starter chuẩn collect 42 tests và ban đầu có thể fail 42 tests. Điều quan trọng lúc này là không có lỗi import, dependency hoặc test collection.

Sau đó mở `exercises.md`, hoàn thành Part 1:

### Exercise 1.1 — Metric thresholds

Với từng metric, viết đủ ba ý: trường hợp score thấp vẫn chấp nhận được, trường hợp score thấp là critical, và hành động sửa. Dựa vào ý nghĩa sau:

- Faithfulness thấp: answer chứa claim không được context hỗ trợ.
- Relevance thấp: answer không giải quyết đúng intent của question.
- Context Recall thấp: retriever bỏ sót evidence cần thiết.
- Context Precision thấp: evidence có thể đã được lấy nhưng bị xếp sau nhiều noise.
- Completeness thấp: answer thiếu các điều kiện/ngoại lệ có trong expected answer.

Không nên đặt mọi threshold giống nhau mà không giải thích. Với safety/privacy, một lỗi nghiêm trọng có thể phải block deployment dù average còn cao.

### Exercise 1.2 — Bias

- Position bias experiment: dùng cùng hai answers A/B, condition 1 đặt A trước B, condition 2 đổi B trước A; giữ nguyên prompt/rubric/model; so sánh mức thay đổi điểm hoặc winner.
- Giảm verbosity bias: rubric chấm theo claim/evidence/điều kiện cần thiết, không thưởng độ dài; phạt thông tin thừa hoặc unsupported.
- Human calibration: lấy tập đã có human labels, đo agreement và chỉnh rubric/judge prompt trước khi tin vào automated scores.

### Exercise 1.3 — CI/CD

Chọn threshold riêng cho Faithfulness, Relevance, Completeness và giải thích theo rủi ro. Trình bày rõ:

- Offline evaluation: chạy trước release/prompt/model/retriever change.
- Online evaluation: theo dõi traffic thật, drift và trải nghiệm người dùng.
- Human review: dùng cho case high-stakes, ambiguous, safety/privacy hoặc để hiệu chỉnh judge.

### Kiểm tra hoàn tất bước 2

- Test suite collect được 42 tests.
- Các bảng và câu hỏi Part 1 trong `exercises.md` không còn trống.

### Làm gì tiếp theo?

Hoàn thiện `template.py` theo từng Task ở bước 3; không viết metric mới vào `evaluate_answers.py`.

## 5. Bước 3 — Hoàn thiện evaluation core trong `template.py`

Không đổi tên class, function hoặc signature. Dùng `_tokenize()` có sẵn, xử lý tập rỗng và clamp mọi score vào `[0.0, 1.0]`.

### 5.1 Task 1 — Data models

Khai báo `QAPair` theo đúng thứ tự và default:

- `question: str`
- `expected_answer: str`
- `context: str = ""`
- `metadata: dict = field(default_factory=dict)`
- `retrieved_contexts: list[str] = field(default_factory=list)`

Khai báo `EvalResult` với:

- `qa_pair`, `actual_answer`
- `faithfulness`, `relevance`, `completeness`
- `passed`, `failure_type`
- `context_precision` và `context_recall` là optional, mặc định `None`

`overall_score()` chỉ lấy trung bình ba answer-side metrics:

```text
(faithfulness + relevance + completeness) / 3
```

Không đưa Context Recall/Precision vào overall score.

Chạy checkpoint:

```powershell
pytest tests/test_solution.py::TestEvalResultOverallScore -v
```

Mong đợi: `3 passed`.

**Xong Task 1 thì làm tiếp Task 2.**

### 5.2 Task 2 — Năm metrics và `run_full_eval`

Ba answer metrics:

```text
Faithfulness = |answer_tokens ∩ context_tokens| / |answer_tokens|
Relevance    = |answer_tokens ∩ question_tokens| / |question_tokens|
Completeness = |answer_tokens ∩ expected_tokens| / |expected_tokens|
```

Quy tắc tập rỗng theo docstring:

- Answer rỗng trong Faithfulness → `1.0`.
- Question rỗng trong Relevance → `1.0`.
- Expected rỗng trong Completeness/Context Recall/Context Precision → `1.0`.

Context Recall:

1. Tokenize từng retrieved chunk.
2. Hợp tất cả token thành `union_tokens`.
3. Tính `|expected_tokens ∩ union_tokens| / |expected_tokens|`.

Context Precision là Average Precision có xét ranking:

1. Với mỗi chunk theo đúng thứ tự, tính coverage so với expected tokens.
2. Chunk relevant nếu coverage `>= relevance_threshold` (mặc định `0.1`).
3. Tại mỗi vị trí relevant, tính `precision@k = relevant_seen / k`.
4. Lấy tổng các precision tại vị trí relevant chia số relevant chunks.
5. Không có chunk hoặc không có chunk relevant → `0.0`.

Trong `run_full_eval()`:

1. Tính ba answer metrics.
2. `passed=True` chỉ khi cả ba score `>= 0.5`.
3. Nếu fail, gán failure type dựa trên metric thấp: Faithfulness → `hallucination`, Relevance → `irrelevant`, Completeness → `incomplete`; nếu nhiều vấn đề ngang nhau, giữ logic nhất quán với mô tả starter.
4. Tạo `QAPair` và `EvalResult`.
5. Nếu `contexts is None`, retrieval fields phải là `None`; nếu có list contexts, tính và lưu hai retrieval scores.

Chạy checkpoint:

```powershell
pytest tests/test_solution.py::TestRAGASEvaluator tests/test_solution.py::TestContextMetrics tests/test_solution.py::TestRetrievalMetricWiring::test_run_full_eval_connects_optional_retrieval_metrics -v
```

Mong đợi phần bắt buộc: `14 passed, 1 skipped`. Test skip là bonus reranking.

**Xong Task 2 thì làm tiếp Task 3.**

### 5.3 Task 3 — `LLMJudge`

Trong `__init__`, lưu callable `judge_llm_fn` vào instance.

Trong `score_response()`:

1. Tạo prompt chứa question, answer, rubric dimensions và yêu cầu output JSON.
2. Gọi đúng một lần `judge_llm_fn(prompt)`.
3. Parse chuỗi JSON bằng `json.loads`.
4. Chuẩn hóa return có ít nhất `scores` và `reasoning`.
5. Nếu mock trả trực tiếp dictionary điểm như `{"accuracy": 0.8}`, đặt nó vào `scores`; reasoning có thể dùng chuỗi mặc định hợp lý.
6. Xử lý output không parse được theo cách an toàn, không để kiểu trả về sai contract.

Trong `detect_bias(scores_batch)`, luôn trả dictionary có ba key:

- `positional_bias`
- `leniency_bias`
- `severity_bias`

Dùng dữ liệu batch để nhận diện xu hướng; batch quá ít thì báo không đủ evidence thay vì khẳng định chắc chắn. Lưu ý tên bias bắt buộc trong test là ba key trên, dù phần lý thuyết còn nhắc verbosity/self-preference.

Chạy:

```powershell
pytest tests/test_solution.py::TestLLMJudge -v
```

Mong đợi: `4 passed`.

**Xong Task 3 thì làm tiếp Task 4.**

### 5.4 Task 4 — `BenchmarkRunner`

`run()` phải lặp qua từng `QAPair`:

1. Gọi `agent_fn(pair.question)` lấy actual answer.
2. Gọi `evaluator.run_full_eval(...)` với question, answer, gold context, expected answer.
3. Bắt buộc truyền `contexts=pair.retrieved_contexts`.
4. Thu các `EvalResult` theo đúng thứ tự input.

`generate_report()` trả đủ:

- `total`, pass count/rate theo contract trong docstring.
- Average Faithfulness, Relevance, Completeness.
- Average Context Recall/Precision chỉ trên các giá trị không phải `None`; nếu hoàn toàn không có retrieval score thì trả `None`.
- Failure type counts.

`run_regression()`:

1. Tính average ba answer metrics cho new và baseline.
2. Một metric regression nếu `baseline_avg - new_avg > 0.05`.
3. Liệt kê tên metric regression.
4. `passed=True` khi danh sách regression rỗng.

`identify_failures()` lọc result nếu **bất kỳ** answer-side metric nào dưới threshold.

Chạy:

```powershell
pytest tests/test_solution.py::TestBenchmarkRunner tests/test_solution.py::TestRunRegression tests/test_solution.py::TestRetrievalMetricWiring::test_runner_forwards_retrieved_contexts tests/test_solution.py::TestRetrievalMetricWiring::test_report_includes_retrieval_averages -v
```

Mong đợi: `11 passed`.

**Xong Task 4 thì làm tiếp Task 5.**

### 5.5 Task 5 — `FailureAnalyzer`

- `categorize_failures()`: đếm theo `failure_type`; xử lý list rỗng.
- `find_root_cause()`: so sánh ba metric và trả đúng một trong các chuỗi root-cause ghi trong docstring.
- `generate_improvement_suggestions()`: dựa vào failure clusters, tạo action cụ thể; với các loại hallucination/irrelevant/incomplete nên có suggestion tương ứng. Test yêu cầu ít nhất 3 suggestions khi có đủ ba nhóm.
- `generate_improvement_log()`: tạo Markdown table, mỗi failure một dòng, ID dạng `F001`, type, root cause, suggested fix và status luôn là `Open`. Nếu suggestions ngắn hơn failures, dùng fallback hợp lý.

Chạy:

```powershell
pytest tests/test_solution.py::TestFailureAnalyzer tests/test_solution.py::TestGenerateImprovementLog -v
```

Mong đợi: `9 passed`.

### 5.6 Kiểm tra toàn bộ core

```powershell
pytest tests/ -v
python template.py
```

Phần bắt buộc hoàn chỉnh phải đạt `41 passed, 1 skipped`. Nếu làm bonus `rerank_by_overlap()`, kết quả là `42 passed`. `python template.py` chỉ là demo, không thay thế pytest.

### Làm gì tiếp theo?

Chỉ khi core pass checkpoint, chuyển sang thiết kế golden dataset ở bước 4.

## 6. Bước 4 — Thiết kế `golden_dataset.json`

Corpus synthetic là nguồn sự thật duy nhất. Không dùng kiến thức ngoài corpus và không sửa corpus để khớp câu trả lời mong muốn.

### 6.1 Phân bổ bắt buộc

- E01–E05: 5 Easy, factual lookup, thường một document.
- M01–M07: 7 Medium, kết hợp process hoặc 2–3 pieces of evidence.
- H01–H05: 5 Hard, nhiều điều kiện, ngoại lệ, date/version hoặc ambiguity.
- A01: out-of-scope.
- A02: prompt injection.
- A03: false premise/ambiguous trap.

Không đổi `id`, `difficulty`, `attack_type`, `schema_version` hoặc `corpus_id`.

### 6.2 Bản đồ 10 documents để phân phối câu hỏi

| File | Nội dung thích hợp để tạo case |
|---|---|
| `00_system_scope.md` | Scope, từ chối, prompt injection, privacy/safety |
| `01_product_catalog.md` | Specs NovaBook/PulsePhone/AeroBuds/HomeHub, compatibility |
| `02_orders_and_payments.md` | Order states, payment, cancellation, installments, address |
| `03_promotions_and_membership.md` | OrbitPlus, discount stacking, bundle, return extension |
| `04_shipping_and_delivery.md` | ETA, delayed trace, damage, missing package, exceptions |
| `05_returns_and_exchanges.md` | Return windows, restocking, hygiene, refunds, exchange |
| `06_warranty_policy.md` | Warranty length, coverage, exclusions, remedies |
| `07_repair_and_technical_support.md` | Safe troubleshooting, repair time, quote, data/loaner |
| `08_accounts_privacy_and_security.md` | Compromise, fraud, authorization, prohibited secrets |
| `09_escalation_and_policy_updates.md` | Escalation, complaint, triggering date, v1/v2 rules |

Dataset phải dùng đủ cả 10 source documents ít nhất một lần.

### 6.3 Cách làm từng record

Với mỗi ID:

1. Chọn một policy/use case chưa bị trùng.
2. Viết question bằng tiếng Anh, không nhét nguyên câu trả lời vào question.
3. Viết expected answer ngắn nhưng đủ amount, date, condition và exception.
4. Copy evidence **nguyên văn** từ đúng file Markdown vào `contexts[].text`.
5. Đặt `source_doc` đúng chính xác tên file.
6. Kiểm từng claim của expected answer: claim nào chưa có evidence thì thêm đoạn evidence hoặc bỏ claim.
7. Với Medium/Hard, chỉ thêm nhiều contexts khi chúng thực sự cần để suy luận; không thêm evidence rác để lấy coverage.

Một kế hoạch coverage hợp lý là dùng Easy để phủ product/order/shipping/warranty/repair; Medium để kết hợp promotions–returns, orders–security, warranty–repair, shipping–escalation; Hard để kiểm tra policy version, bundle exceptions, account compromise/order state; ba Adversarial dùng scope và khi cần thêm security/policy evidence.

### 6.4 Chú ý cho ba adversarial cases

- A01 phải yêu cầu ngoài phạm vi OrbitTech; expected answer giới hạn vai trò và gợi ý topic được hỗ trợ.
- A02 phải chứa instruction cố ghi đè rule/reveal prompt/secret/data; expected answer bỏ qua instruction và không tiết lộ.
- A03 phải có premise sai hoặc thiếu dữ kiện quan trọng; expected answer không đoán, nêu các khả năng và yêu cầu thông tin cần thiết.
- Cả ba phải có evidence phù hợp từ `00_system_scope.md` theo validator.

### Kiểm tra hoàn tất bước 4

Chạy liên tục trong lúc điền:

```powershell
python validate_golden_dataset.py
```

Chỉ hoàn tất khi thấy:

```text
PASS: dataset structure and evidence provenance are valid.
```

Validator chỉ xác nhận schema/provenance/coverage, không đảm bảo difficulty hay semantic quality. Sau PASS, tự review 20 records rồi điền bảng Exercise 3.1 trong `exercises.md`: 20/20, 5/7/5/3, 10/10 sources, status PASS, ba case đại diện và quyết định thiết kế.

### Làm gì tiếp theo?

Sau validator PASS, cấu hình API và sinh actual answers ở bước 5. Nếu chưa PASS, tuyệt đối chưa gọi API.

## 7. Bước 5 — Sinh 20 actual answers bằng RAG thật

Kiểm tra `.env` có:

```dotenv
OPENAI_API_KEY=<key thật>
OPENAI_MODEL=gpt-4o-mini
```

Không in hoặc commit key. Sau đó chạy:

```powershell
python domain_assistant.py
```

Script dùng mặc định corpus hiện tại, golden dataset, `top-k 5`, và ghi `artifacts/actual_answers.json`. Nó chỉ đọc `id` và `question` để generation; không được sửa để đọc `expected_answer` vì đó là data leakage.

Mở artifact và kiểm đủ 20 IDs:

- Question khớp golden dataset.
- `actual_answer` không rỗng.
- `retrieved_contexts` có source, chunk ID, text, score.
- `error` là `null`.

Nếu golden dataset thay đổi sau lần sinh, phải validate và chạy lại `domain_assistant.py`.

### Làm gì tiếp theo?

Khi artifact đủ và không lỗi, chạy evaluation ở bước 6.

## 8. Bước 6 — Chạy benchmark thật và điền Exercise 3.2

Điều kiện: core đã pass, dataset đã PASS, actual answers đủ 20. Chạy:

```powershell
python evaluate_answers.py
```

Kết quả được lưu tại `artifacts/benchmark_results.json`. Dùng bảng terminal hoặc artifact để điền toàn bộ 20 dòng Exercise 3.2:

- Context Recall
- Context Precision
- Faithfulness
- Relevance
- Completeness
- Overall
- Passed
- Failure Type

Điền aggregate report và sắp xếp để lấy ba Overall Score thấp nhất.

Cách đọc nguyên nhân:

- Recall thấp + Completeness thấp → retrieval có thể bỏ sót evidence.
- Recall cao + Precision thấp → đã lấy evidence nhưng ranking/noise kém.
- Retrieval tốt + Faithfulness thấp → generator thêm claim ngoài context.
- Faithfulness cao + Relevance thấp → answer grounded nhưng sai intent.
- Retrieval tốt + Completeness thấp → generation bỏ sót nội dung cần thiết.

Benchmark score không quyết định điểm lab; chất lượng pipeline, evidence và phân tích mới là tiêu chí chính.

### Làm gì tiếp theo?

Thiết kế rubric Exercise 3.3 rồi dùng ba case thấp nhất để viết reflection.

## 9. Bước 7 — Hoàn thành Exercise 3.3

Chọn 3–5 dimensions phù hợp OrbitTech, ví dụ Correctness, Completeness, Evidence, Actionability, Safety/Privacy. Viết tiêu chí rõ cho từng mức 1–5:

- 5: đúng, đủ điều kiện/ngoại lệ, grounded, actionable, an toàn, không tiết lộ dữ liệu.
- 4: đúng phần cốt lõi nhưng thiếu một chi tiết nhỏ không làm đổi hành động.
- 3: hữu ích một phần nhưng thiếu điều kiện quan trọng hoặc hơi mơ hồ.
- 2: có ít nội dung đúng nhưng hướng dẫn sai/thiếu nghiêm trọng hoặc unsupported.
- 1: sai bản chất, hallucination, vi phạm safety/privacy hoặc làm theo prompt injection.

Đây chỉ là khung; phải cụ thể hóa bằng ví dụ response thuộc domain. Điền thêm ba edge cases và bias controls. Rubric không được thưởng answer chỉ vì dài.

### Làm gì tiếp theo?

Mở `reflection.md` và làm bước 8.

## 10. Bước 8 — Viết `reflection.md`

### 10.1 Benchmark summary

Copy aggregate metrics, số pass/fail, failure distribution và nhận xét metric yếu nhất.

### 10.2 Ba failure tệ nhất và 5 Whys

Với mỗi case:

1. Ghi ID, question, expected answer, actual answer và năm metric.
2. So sánh gold evidence với retrieved chunks.
3. Nêu symptom quan sát được, không nhảy ngay tới root cause.
4. Hỏi “Why?” năm lần; mỗi câu trả lời phải dẫn hợp lý đến câu tiếp.
5. Kết thúc bằng root cause có thể hành động, ví dụ chunking/ranking/prompt/guardrail.
6. So sánh với output của `FailureAnalyzer.find_root_cause()`.
7. Đề xuất fix cụ thể và metric/check dùng để verify.

### 10.3 Failure clustering và improvement log

Gom failure theo hallucination/irrelevant/incomplete/off-topic/refusal hoặc theo retrieval/generation/policy handling. Ưu tiên fix một root cause giải quyết nhiều cases. Điền bảng improvement log với owner/action/status/verification metric phù hợp form.

### 10.4 Regression strategy

Nêu rõ:

- Baseline artifact/version nào được lưu.
- Khi nào CI chạy evaluation.
- Rule hiện có: metric average giảm hơn `0.05` là regression.
- Safety/privacy critical case có thể block riêng, không chỉ dựa average.
- Cách xử lý nondeterminism: fixed dataset/settings, nhiều lần chạy hoặc human review cho case dao động.

### 10.5 Continuous improvement và final reflection

Mô tả vòng lặp Evaluate → Analyze → Improve → Augment → Repeat bằng kết quả thật của bài, không viết lý thuyết chung chung.

### Làm gì tiếp theo?

Rà lại worksheet và tạo file nộp `solution/solution.py`.

## 11. Bước 9 — Tạo solution và kiểm tra cuối

Copy core đã hoàn thiện:

```powershell
Copy-Item template.py solution/solution.py -Force
```

Sau khi file này tồn tại, pytest sẽ load nó thay vì `template.py`. Chạy:

```powershell
pytest tests/ -v
python validate_golden_dataset.py
python evaluate_answers.py
git diff --check
git status --short
```

Mục tiêu bắt buộc:

- `41 passed, 1 skipped` (hoặc 42 passed nếu làm bonus reranking).
- Validator báo PASS.
- Benchmark chạy xong không có lỗi.
- Không có whitespace error nghiêm trọng từ `git diff --check`.

Nếu sửa `template.py` sau bước copy, chạy lại `Copy-Item ... -Force` trước khi test.

## 12. Checklist nộp bài

- [ ] `solution/solution.py` có đầy đủ implementation bắt buộc.
- [ ] Required tests pass; không sửa tests để ép pass.
- [ ] `golden_dataset.json` có đúng 20 QA theo 5 Easy + 7 Medium + 5 Hard + 3 Adversarial.
- [ ] Dataset dùng đủ 10 documents và mọi evidence là trích nguyên văn.
- [ ] Validator báo PASS.
- [ ] `exercises.md` hoàn thành Part 1, Exercise 3.1, 3.2 và 3.3.
- [ ] Exercise 3.2 có đủ năm metrics, aggregate report và ba case thấp nhất.
- [ ] `reflection.md` có ba 5 Whys, failure clustering, improvement log và regression strategy.
- [ ] `.env` và API key không nằm trong commit/diff.
- [ ] Không sửa corpus hoặc cho RAG đọc gold answer để tăng benchmark.
- [ ] Bonus chỉ làm sau khi phần bắt buộc hoàn tất.

## 13. Xử lý nhanh các lỗi thường gặp

| Lỗi | Cách xử lý |
|---|---|
| `ModuleNotFoundError` | Activate `.venv`, cài lại `requirements.txt` |
| Validator báo field rỗng | Điền đủ question/expected/context cho 20 slot |
| `text is not a verbatim substring` | Copy lại đúng nguyên văn, kể cả punctuation và spacing |
| Thiếu source coverage | Thiết kế case thật sự liên quan cho document còn thiếu |
| `OPENAI_API_KEY is missing` | Kiểm `.env`, key placeholder và current directory |
| `question differs between artifacts` | Dataset đã đổi; chạy lại generation |
| `Complete required TODOs` | Quay lại targeted checkpoint tương ứng |
| Sửa template nhưng test không đổi | `solution/solution.py` đang được ưu tiên; copy lại template |
| Retrieval metrics là `None` | Kiểm trace trong artifact, `retrieved_contexts`, và Runner truyền `contexts` |
| Test fail hàng loạt | Chạy đúng một test class/function với `-v`, sửa lỗi đầu tiên trước |

## 14. Bonus (chỉ sau khi đạt phần bắt buộc)

- Exercise 3.4: so sánh hai framework trên cùng dataset/input, nêu setup, metrics, ưu/nhược và kết quả.
- Exercise 3.5: implement `rerank_by_overlap()`, giữ nguyên tập chunks, đo Context Precision trước/sau trên ít nhất năm traces. Reranking chỉ đổi thứ tự nên union coverage/Context Recall về lý thuyết không đổi.

Thứ tự an toàn cuối cùng là: **core pass → dataset PASS → sinh answers → benchmark → exercises/reflection → copy solution → final tests**.
