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
| Faithfulness | Answer diễn giải/tóm tắt lại context bằng từ ngữ khác nên overlap từ vựng thấp dù ý nghĩa vẫn đúng | Answer thêm thông tin không có trong context (giá, chính sách hoàn tiền, điều kiện bảo hành bịa ra) | Dưới 0.8: audit generation prompt, kiểm tra hallucination systematic, có thể cần thêm constraint "chỉ trả lời dựa trên context" |
| Answer Relevance | Câu hỏi mơ hồ, có nhiều cách diễn giải hợp lệ nên answer lệch nhẹ khỏi trọng tâm | Answer lạc đề hoàn toàn, không giải quyết intent của khách hàng (hỏi A trả lời B) | Dưới 0.6: review lại question understanding / intent detection trong pipeline |
| Context Recall | Câu hỏi cần tổng hợp nhiều docs, thiếu một chunk phụ không ảnh hưởng câu trả lời chính | Thiếu chunk chứa thông tin cốt lõi (vd điều kiện đổi trả), khiến answer chắc chắn sai hoặc thiếu | Dưới 0.6: investigate retriever — tăng top-K, cải thiện chunking hoặc embedding |
| Context Precision | Có vài noise chunk ở cuối danh sách nhưng top-ranked chunk vẫn đúng | Chunk relevant xếp hạng thấp hoặc chunk đầu toàn irrelevant, khiến generation dựa trên context sai | Dưới 0.6: investigate ranking/reranking, kiểm tra query formulation |
| Completeness | Answer thiếu chi tiết phụ, không ảnh hưởng đến việc giải quyết yêu cầu chính của khách hàng | Answer thiếu bước hoặc điều kiện bắt buộc (vd thiếu yêu cầu "cần hóa đơn" khi đổi trả) | Dưới 0.6: analyze failures — bổ sung instruction yêu cầu liệt kê đủ điều kiện/bước trong prompt |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy cùng một cặp answer (A, B) cho cùng một câu hỏi, cho judge chấm ở hai conditions đảo vị trí:
> - Condition 1: trình bày (A, B) — A đứng trước.
> - Condition 2: trình bày (B, A) — B đứng trước.
>
> Lặp lại trên nhiều cặp câu hỏi/câu trả lời khác nhau, đo tỷ lệ judge chọn "answer đứng trước" ở cả hai conditions. Nếu tỷ lệ này lệch đáng kể khỏi 50% (và không phụ thuộc nội dung answer), tức là judge có position bias. Có thể mở rộng thêm condition 3: hai answer chất lượng ngang nhau (paraphrase của nhau) để cô lập ảnh hưởng của vị trí khỏi ảnh hưởng của nội dung.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Viết rubric nêu rõ tiêu chí chấm là độ chính xác/mức độ đáp ứng intent, không phải độ dài — ví dụ ghi rõ "answer ngắn nhưng đủ ý vẫn đạt điểm tối đa; answer dài dòng, lặp ý không được cộng điểm". Kèm ví dụ cụ thể ở mỗi mức điểm cho thấy một answer ngắn có thể đạt 5đ và một answer dài có thể chỉ đạt 2đ nếu sai/lan man, để judge có anchor thay vì suy luận theo độ dài. Có thể thêm rule phạt trực tiếp: nội dung thừa không đóng góp thông tin mới bị trừ điểm ở dimension "Conciseness/Relevance".

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Vì LLM judge có thể mang các bias hệ thống (position, verbosity, self-preference) hoặc đánh giá lệch chuẩn so với con người mà bản thân judge không tự nhận ra. Calibration là so sánh score của judge với score của human trên một tập mẫu (agreement rate, correlation), từ đó biết judge có đáng tin cậy không, ở mức nào thì lệch nhiều, và điều chỉnh rubric/threshold trước khi dùng judge để quyết định ở quy mô lớn (vd gate CI/CD). Nếu không calibrate, một judge bias có thể âm thầm pass/fail sai hàng loạt case mà không ai phát hiện.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.8 | Hallucination gây thông tin sai trực tiếp cho khách hàng (giá, chính sách hoàn tiền, bảo hành) — rủi ro cao nhất nên cần threshold chặt nhất |
| Answer Relevance | 0.7 | Answer lạc đề gây trải nghiệm xấu và mất thời gian khách hàng, nhưng ít nguy hiểm hơn hallucination nên threshold thấp hơn một chút |
| Completeness | 0.65 | Thiếu chi tiết phụ chấp nhận được ở mức độ nhất định, nhưng thiếu bước/điều kiện bắt buộc thì không nên vẫn cần threshold đủ cao để chặn case thiếu thông tin quan trọng |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation**: chạy trên golden dataset cố định trước khi deploy, dùng để gate CI/CD — phát hiện regression trước khi code lên production, chi phí thấp và lặp lại được.
> - **Online evaluation**: theo dõi trên real traffic sau khi deploy (A/B test, sampling một phần request thật), dùng để phát hiện drift hoặc các case thực tế không có trong golden dataset mà offline eval không cover được.
> - **Human review**: dùng khi automated score nằm gần threshold (borderline case), khi phát hiện failure type mới lạ chưa từng thấy, hoặc định kỳ để calibrate lại LLM judge/metrics với nhãn con người — vì automated metrics không thể thay thế hoàn toàn đánh giá con người ở các case nhạy cảm hoặc mơ hồ.

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
