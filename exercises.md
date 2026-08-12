# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Score thấp có thể tạm chấp nhận khi câu trả lời chỉ bổ sung lời chào, cách diễn đạt tự nhiên hoặc hướng dẫn chung an toàn không làm thay đổi nội dung chính sách. | Score thấp là critical khi câu trả lời bịa thông số sản phẩm, trạng thái đơn hàng, mức giảm giá, thời hạn, quyền lợi bảo hành hoặc đưa ra claim không có trong corpus; đặc biệt nghiêm trọng với privacy và safety. | Kiểm tra gold/retrieved contexts, cải thiện retrieval và system prompt; yêu cầu mọi policy claim phải được grounded, thêm hallucination guardrail và block deployment nếu lỗi safety/privacy tái diễn. |
| Answer Relevance | Score thấp có thể chấp nhận khi câu hỏi mơ hồ và assistant chủ động hỏi lại thông tin cần thiết trước khi trả lời, chẳng hạn order date hoặc order status. | Score thấp là critical khi câu hỏi rõ ràng nhưng assistant trả lời sai intent, chuyển sang một policy không liên quan hoặc đưa hướng dẫn không giải quyết nhu cầu của khách hàng. | Cải thiện intent detection và prompt, thêm examples cho các intent dễ nhầm, kiểm tra routing/retrieval query và bổ sung test cases cho câu hỏi mơ hồ. |
| Context Recall | Score thấp có thể chấp nhận khi expected answer chứa nhiều cách diễn đạt tương đương nhưng retrieved chunks vẫn có đủ evidence cốt lõi để tạo câu trả lời đúng và an toàn. | Score thấp là critical khi retriever bỏ sót điều kiện, ngoại lệ, effective date hoặc policy version làm câu trả lời có thể sai; ví dụ thiếu quy định v1/v2 của return policy. | Kiểm tra chunking và truy vấn BM25, điều chỉnh top_k, bổ sung metadata/filter hoặc query expansion; thêm case bị bỏ sót vào regression suite và đo lại Context Recall. |
| Context Precision | Score thấp có thể chấp nhận khi evidence đúng vẫn nằm trong top-k và context window đủ lớn, nên generator vẫn nhận được đầy đủ thông tin dù có một số chunk nhiễu. | Score thấp là critical khi nhiều chunk không liên quan đứng trước evidence, làm evidence quan trọng bị đẩy khỏi context window hoặc khiến model trộn lẫn các policy. | Áp dụng reranking, cải thiện retrieval query/chunking, lọc chunk theo document/intent và so sánh Context Precision trước–sau mà vẫn theo dõi Context Recall. |
| Completeness | Score thấp có thể chấp nhận khi assistant cố ý trả lời ngắn, nhưng vẫn cung cấp kết luận và hành động chính; phần thiếu chỉ là chi tiết phụ không ảnh hưởng quyết định của khách hàng. | Score thấp là critical khi câu trả lời bỏ sót amount, deadline, eligibility, exception, required evidence hoặc bước an toàn, khiến khách hàng có thể thực hiện sai quy trình. | Viết prompt yêu cầu kiểm tra đủ điều kiện và ngoại lệ, cải thiện retrieval nếu evidence bị thiếu, thêm answer checklist/few-shot examples và tạo regression tests cho các chi tiết thường bị bỏ sót. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> Lấy cùng một cặp answer (A, B) cho mỗi câu hỏi trong golden dataset, rồi cho
> judge chấm **hai lần** với thứ tự trình bày đảo ngược:
> - **Condition 1 (A-first):** prompt đưa answer A trước, answer B sau, hỏi judge chọn cái tốt hơn.
> - **Condition 2 (B-first):** cùng cặp A/B nhưng đảo vị trí — B trước, A sau.
>
> Giữ nguyên nội dung answer, chỉ đổi thứ tự xuất hiện. Sau đó tính:
> - **Flip rate**: tỷ lệ % câu hỏi mà lựa chọn "answer tốt hơn" của judge đổi
>   theo vị trí thay vì theo nội dung (judge chọn A ở condition 1 nhưng lại
>   chọn B ở condition 2 dù nội dung B ở condition 2 chính là A ở condition 1).
> - **First-position win rate**: tỷ lệ judge chọn answer ở vị trí đầu tiên,
>   gộp cả hai condition. Nếu tỷ lệ này lệch đáng kể khỏi 50% (ví dụ >65%),
>   đó là bằng chứng thống kê của position bias.
> - Để chắc chắn hơn, chạy thêm với cặp answer chất lượng ngang nhau (đã biết
>   trước là "tie") — nếu judge vẫn thiên vị theo vị trí thay vì trả về tie,
>   bias càng rõ.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Thêm tiêu chí **conciseness** rõ ràng vào rubric (ví dụ: "trừ điểm nếu câu
>   trả lời chứa thông tin thừa, lặp lại hoặc padding không cần thiết"), thay
>   vì chỉ có các tiêu chí "đầy đủ/chi tiết" vốn vô tình thưởng cho câu dài.
> - Tách rõ **completeness** (đủ ý cần thiết theo `expected answer`/gold
>   context) khỏi **length** — yêu cầu judge chấm completeness dựa trên số ý
>   bắt buộc có mặt, không dựa trên số câu/số từ.
> - Đưa **few-shot calibration examples** vào prompt của judge: một cặp câu
>   trả lời ngắn-đúng-đủ ý vs. câu trả lời dài-dư thừa-không thêm giá trị, kèm
>   điểm mẫu, để judge học chuẩn "ngắn mà đủ vẫn là điểm cao nhất".
> - Chỉ thị tường minh trong prompt: "Không được cho điểm cao hơn chỉ vì câu
>   trả lời dài hơn; đánh giá dựa trên độ chính xác và mức độ đủ ý so với
>   yêu cầu, không dựa trên độ dài."
> - Nếu dùng pairwise comparison, có thể chuẩn hoá độ dài giữa hai answer
>   trước khi đưa cho judge (cắt bớt/prompt answer ngắn hơn viết dài tương
>   đương) để tách biệt effect của length khỏi effect của quality.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> LLM judge có thể mắc đúng các bias nêu trên (position, verbosity,
> self-preference) và không có gì đảm bảo "cảm nhận đúng/sai" của nó khớp với
> tiêu chuẩn nghiệp vụ thực tế — ví dụ chính sách đổi trả, bảo hành của
> OrbitTech có các ràng buộc rất cụ thể mà một judge tổng quát có thể đánh giá
> sai dù văn phong answer nghe "hợp lý". Calibrate bằng cách lấy một tập nhỏ
> câu hỏi đã có nhãn con người (human label: đúng/sai, hoặc thang điểm), cho
> LLM judge chấm cùng tập đó, rồi đo mức độ đồng thuận (agreement rate, Cohen's
> kappa, hoặc correlation giữa điểm judge và điểm người). Nếu đồng thuận thấp,
> cần điều chỉnh rubric/prompt của judge rồi đo lại, lặp lại đến khi đạt
> ngưỡng tin cậy chấp nhận được. Không calibrate thì không có cách nào biết
> điểm số tự động của judge có phản ánh đúng chất lượng thật hay chỉ phản ánh
> bias/thiên hướng riêng của model đó — dẫn đến quyết định sai (ví dụ chặn
> deploy oan, hoặc bỏ lọt regression thật).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | ≥ 0.85 | Faithfulness thấp nghĩa là câu trả lời không grounded vào context — rủi ro bịa chính sách, giá, bảo hành. Hậu quả trực tiếp tới khách hàng và pháp lý nên đặt cao hơn ngưỡng "good" chung (0.8) thay vì chỉ chặn ở mức "significant issues" (0.6). |
| Answer Relevance | ≥ 0.75 | Nằm ở biên trên của dải "needs work" (0.6–0.8): câu trả lời lạc đề vẫn gây trải nghiệm xấu nhưng ít nguy hiểm hơn hallucination, nên có thể cho qua nếu vượt 0.75 và theo dõi thêm thay vì chặn cứng ở 0.8. |
| Completeness | ≥ 0.75 | Thiếu chi tiết phụ (câu hỏi hẹp) đôi khi chấp nhận được, nhưng dưới 0.75 có nguy cơ bỏ sót amount/deadline/exception quan trọng khiến khách hàng làm sai quy trình — đủ nghiêm trọng để chặn deploy. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation** (golden dataset, chạy trong CI/CD trước khi merge/deploy):
>   dùng làm **gate tự động** — nhanh, lặp lại được, cùng một bộ câu hỏi/threshold
>   mỗi lần nên phát hiện regression sớm trước khi code lên production. Phù hợp
>   để chặn deploy khi Faithfulness/Answer Relevance/Completeness rớt dưới
>   threshold ở Câu 1.
>   Giới hạn: golden dataset chỉ 20 câu, không phủ hết cách hỏi thật của khách
>   hàng ngoài production.
> - **Online evaluation** (giám sát traffic thật sau khi đã deploy): dùng để
>   **theo dõi liên tục** — lấy mẫu câu hỏi/câu trả lời thật, chấm bằng LLM
>   judge hoặc RAGAS batch định kỳ, theo dõi drift theo thời gian (retrieval
>   xuống cấp khi corpus đổi, model provider update...). Không chặn từng
>   deploy nhưng dùng để cảnh báo (alert) khi metric tụt dưới threshold trên
>   traffic thật — bắt được case golden dataset không có.
> - **Human review**: dùng khi (1) calibrate LLM judge với nhãn người (theo
>   Exercise 1.2 Câu 3) trước khi tin tưởng threshold tự động; (2) score nằm
>   sát biên threshold (borderline, ví dụ 0.72–0.78) mà automated judge không
>   đủ tin cậy để quyết định pass/fail một mình; (3) câu hỏi thuộc nhóm rủi ro
>   cao — an toàn, quyền riêng tư, khiếu nại pháp lý — luôn cần người xác nhận
>   dù score tự động cao; (4) audit định kỳ lấy mẫu ngẫu nhiên từ production để
>   phát hiện lỗi hệ thống mà cả offline lẫn online evaluation đều bỏ sót.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
