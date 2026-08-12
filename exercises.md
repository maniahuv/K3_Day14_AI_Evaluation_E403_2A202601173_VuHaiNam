# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi yêu cầu từ chối (adversarial out-of-scope) hoặc model trả lời đúng nhờ zero-shot knowledge dù context thiếu (vẫn không faithful với context nhưng đúng thực tế). | Sinh ra thông tin sai lệch (hallucination) hoặc trực tiếp mâu thuẫn với context, đặc biệt là các thông tin quan trọng như deadline, học phí. | Cải thiện prompt để model bám sát context (grounding). Nếu thiếu thông tin ở context, cải thiện retrieval. |
| Answer Relevance | Câu hỏi chung chung, model trả lời một cách bao quát, có chứa thông tin thừa so với core intent nhưng không sai. | Câu trả lời đi lạc đề hoàn toàn, trả lời một câu hỏi khác hoặc cung cấp thông tin không giải quyết được vấn đề của user. | Tinh chỉnh system prompt để tập trung trả lời đúng trọng tâm. Phân tích lỗi phân loại intent. |
| Context Recall | Một chunk duy nhất đã đủ trả lời trọn vẹn câu hỏi, nhưng hệ thống tính điểm dựa trên nhiều chunks (union), làm coverage bị thấp giả tạo. | Retriever bỏ sót hoàn toàn tài liệu/chunk quan trọng nhất chứa đáp án, khiến model không thể trả lời hoặc phải hallucinate. | Cải thiện embedding model, tinh chỉnh chunking strategy (kích thước, overlap) hoặc áp dụng hybrid search. |
| Context Precision | Chunk quan trọng bị xếp hạng thấp (ví dụ top 4, 5) nhưng context window vẫn đủ lớn để LLM đọc và trả lời đúng. | Các chunks không liên quan chiếm hết các vị trí top đầu, đẩy chunk quan trọng ra khỏi context window hoặc làm LLM bối rối. | Áp dụng hoặc tinh chỉnh mô hình Reranking (ví dụ: Cross-Encoder) để đẩy relevant chunks lên vị trí cao nhất. |
| Completeness | Câu trả lời chính xác, đi thẳng vào vấn đề nhưng bỏ qua một vài chi tiết nhỏ không mang tính quyết định có trong gold answer. | Câu trả lời bỏ sót các điều kiện bắt buộc (conditions), ngoại lệ (exceptions) hoặc các bước quan trọng dẫn đến user hành động sai. | Cập nhật prompt yêu cầu liệt kê rõ các điều kiện/ngoại lệ, hoặc tối ưu context recall nếu thông tin đó không được retrieve. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> Condition 1 (Forward): Cung cấp cho LLM Judge hai câu trả lời, Answer A ở vị trí 1 và Answer B ở vị trí 2. Ghi nhận kết quả (A hay B thắng).
> Condition 2 (Swap): Đảo ngược vị trí, Answer B ở vị trí 1 và Answer A ở vị trí 2. Ghi nhận kết quả.
> Nếu LLM Judge có order bias, nó sẽ có xu hướng luôn chọn answer ở vị trí 1 (hoặc luôn chọn vị trí 2) bất kể nội dung, dẫn đến kết quả mâu thuẫn giữa hai conditions. Tỉ lệ không nhất quán (inconsistency rate) chính là mức độ position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> Thiết kế rubric tập trung vào "mật độ thông tin" (information density) thay vì độ dài.
> - Định nghĩa rõ: "Một câu trả lời ngắn gọn, đi thẳng vào vấn đề phải được điểm cao hơn một câu dài dòng nhưng chứa cùng lượng thông tin cốt lõi".
> - Thêm tiêu chí phạt (penalty) cho thông tin thừa thãi (fluff) hoặc không liên quan.
> - Hạn chế yêu cầu giải thích trừ khi cần thiết, vì giải thích dễ làm model ưu tiên câu trả lời dài.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> LLM Judge không phải lúc nào cũng hiểu đúng ngữ cảnh, sắc thái của domain cụ thể, và có thể mang các biases riêng. Calibrate với human labels (bằng cách tính độ tương quan - correlation, agreement rate) giúp:
> 1. Đảm bảo LLM đánh giá đúng định nghĩa về "chất lượng" của con người.
> 2. Phát hiện và điều chỉnh các bias (strictness, verbosity).
> 3. Tìm ra threshold phù hợp để tin tưởng giao phó cho LLM Judge chạy tự động trong CI/CD.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.90 | Hallucination là rủi ro lớn nhất (đặc biệt trong student services với các thông tin nhạy cảm như học phí, deadline). Cần strict threshold để chặn thông tin sai. |
| Answer Relevance | 0.75 | Có thể linh hoạt hơn vì câu trả lời chứa thông tin thừa có thể bị giảm điểm relevance nhưng không gây hại nghiêm trọng như unfaithful. |
| Completeness | 0.85 | Cần đảm bảo sinh viên nhận đủ các điều kiện/ngoại lệ quan trọng, nhưng có thể chấp nhận thiếu một số chi tiết phụ. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation:** Dùng trong CI/CD pipeline, trước khi deploy (development phase). Chạy LLM Judge tự động trên các golden datasets để chống regression và so sánh các phiên bản prompts/models.
> - **Online evaluation:** Dùng sau khi đã deploy (production). Theo dõi user feedback (explicit như thumbs up/down, implicit như click rate), và có thể chạy LLM Judge trên một sample traffic thật để monitor chất lượng realtime (shadow metrics).
> - **Human review:** Dùng để tạo golden dataset ban đầu, định kỳ calibrate lại LLM Judge, và để xử lý các edge cases phức tạp/high-risk mà tự động hóa không thể đánh giá chính xác.

---

## Part 2 — Core Coding (09:45–10:40)

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

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

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

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

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.

