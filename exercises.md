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
| Faithfulness | Khi câu hỏi là giao tiếp xã giao/chào hỏi hoặc nằm ngoài scope khiến bot trả lời theo câu từ chối tiêu chuẩn không chứa thông tin thực thể; hoặc khi diễn đạt lại (paraphrase) dùng từ đồng nghĩa khiến string matching đánh giá thấp dù không hallucinate. | Khi thông tin quan trọng của OrbitTech (giá sản phẩm, thời hạn bảo hành, phí hoàn tiền 20 USD, ngày áp dụng chính sách) bị bot bịa đặt (hallucinate) sai lệch hoàn toàn so với retrieved context. | Tinh chỉnh System Prompt với hướng dẫn nghiêm ngặt ("Chỉ trả lời dựa trên context được cung cấp"), đặt `temperature=0.0`, ép bot từ chối khi context không đủ dữ liệu. |
| Answer Relevance | Khi câu hỏi của người dùng mơ hồ, thiếu dữ kiện và bot chủ động hỏi lại để làm rõ (clarification question), hoặc khi bot chủ động từ chối câu hỏi vi phạm an toàn/out-of-scope. | Khi câu hỏi rõ ràng (ví dụ: "Chính sách bảo hành laptop Asus?") nhưng bot trả lời lan man sang quy trình giao hàng hoặc thông tin không liên quan, không giải quyết intent của người dùng. | Tối ưu System Prompt tập trung vào Intent Extraction, điều chỉnh retriever lấy đúng chunk trọng tâm, bổ sung few-shot ví dụ câu trả lời trực diện. |
| Context Recall | Khi câu hỏi đơn giản/lookup và retriever lấy được đoạn trích nhỏ chứa đúng thông tin cốt lõi, dù không phủ hết toàn bộ đoạn văn phụ xung quanh trong gold context. | Khi với câu hỏi phức tạp (multi-doc/multi-step như phí hủy đơn + điều kiện hoàn tiền), retriever bỏ sót các document/chunk chứa điều kiện quan trọng khiến câu trả lời bị thiếu ý nghiêm trọng. | Chuyển retriever sang Hybrid Search (BM25 + Dense Vector Search), tăng `top_k`, tinh chỉnh chiến lược chunking (dùng chunking overlap hoặc parent-child chunking). |
| Context Precision | Khi `top_k` trả về 5 chunks và các chunk liên quan nằm ở vị trí 2, 3 thay vì vị trí 1, nhưng LLM vẫn lọc thông tin nhiễu tốt và trả lời chính xác. | Các chunk chứa câu trả lời thực sự bị xếp ở cuối danh sách (vị trí thấp) trong khi các chunk rác/không liên quan nằm ở đầu, khiến LLM bị xao lãng ("Lost in the Middle") hoặc trôi mất thông tin. | Bổ sung bước Reranking (dùng Reranker model như Cross-Encoder/Cohere Rerank) sau bước retrieval để xếp các chunk có độ tương quan cao nhất lên đầu. |
| Completeness | Khi người dùng chỉ hỏi một ý hẹp và bot trả lời trực tiếp ý đó mà không liệt kê thêm các quy trình chi tiết chưa được hỏi đến. | Câu hỏi yêu cầu đầy đủ điều kiện/phí phạt (ví dụ: "Điều kiện hủy đơn prepaid và chi phí không hoàn lại?") nhưng bot bỏ quên thông tin phí xử lý 20 USD không hoàn lại, gây thiệt hại cho khách hàng. | Thiết kế checklist/rubric câu trả lời trong Prompt, yêu cầu LLM kiểm tra xem đã bao phủ hết các điều kiện/exceptions chưa, chuẩn hóa Expected Answer trong Golden Dataset. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> 
> - **Mục tiêu**: Kiểm tra xem LLM Judge có xu hướng ưu tiên chọn đáp án đứng ở vị trí A (trước) hơn vị trí B (sau) trong bài toán so sánh cặp (Pairwise Comparison) hay không.
> - **Thiết kế Experiment**:
>   - **Tập dữ liệu**: Chọn 30 cặp câu trả lời $(R_1, R_2)$ từ hai phiên bản model cho cùng một tập câu hỏi trong Golden Dataset.
>   - **Condition 1 (Original Order)**: Đưa vào Prompt của LLM Judge với cấu trúc: `Option A: R_1`, `Option B: R_2`. Ghi lại số lần $R_1$ được chọn thắng (Win Rate $W_1$).
>   - **Condition 2 (Swapped Order)**: Đảo ngược vị trí trong Prompt: `Option A: R_2`, `Option B: R_1`. Ghi lại số lần $R_1$ (lúc này ở vị trí Option B) được chọn thắng (Win Rate $W_2$).
> - **Phân tích & Kết luận**: So sánh $W_1$ và $W_2$. Nếu $W_1$ chênh lệch đáng kể so với $W_2$ (ví dụ: $W_1 = 70\%$ nhưng $W_2 = 30\%$), chứng tỏ Judge bị ảnh hưởng nặng nề bởi **Position Bias** (ưu tiên đáp án đứng ở Option A).
> - **Cách khắc phục**: Thực hiện chấm cả 2 chiều (position swapping) rồi lấy trung bình kết quả, hoặc chuyển sang dùng Point-based Single-answer Grading kèm Rubric chi tiết thay vì Pairwise Comparison.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 
> Để giảm thiểu tình trạng LLM Judge ưu tiên cho điểm cao những câu trả lời dài dòng/hoa mỹ dù chứa thông tin thừa hoặc thiếu chính xác, Rubric Design cần áp dụng các kỹ thuật sau:
> 1. **Thêm tiêu chí Conciseness & Directness bắt buộc**: Đưa yêu cầu "Trả lời đúng trọng tâm, không lan man" vào Rubric. Thêm quy tắc trừ điểm rõ ràng nếu câu trả lời chứa thông tin thừa không liên quan đến câu hỏi.
> 2. **Chuyển sang dạng Point-based Fact Checklist (Granular Scoring)**: Thay vì yêu cầu Judge cho điểm tổng thể 1–5, chia nhỏ Rubric thành danh sách các sự thật cần kiểm tra (Yes/No Checklist):
>    - *Check 1*: Trả lời đúng thời hạn (Có/Không: +1đ)
>    - *Check 2*: Nêu đúng phí 20 USD không hoàn lại (Có/Không: +1đ)
>    - *Check 3*: Chứa thông tin không liên quan / dông dài (Có/Không: -1đ)
> 3. **Đặt giới hạn độ dài kỳ vọng (Length Constraints)**: Nêu rõ trong Prompt của Judge: *"Một câu trả lời tối ưu cần ngắn gọn (dưới 150 từ). Việc bổ sung các câu từ xã giao dài dòng không giúp tăng điểm và sẽ bị trừ điểm nếu làm loãng ý chính."*

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 
> Việc Calibrate (hiệu chỉnh) LLM Judge với Human Labels (đánh giá của chuyên gia/con người) là bắt buộc vì các lý do sau:
> 1. **Đảm bảo Alignment với domain kiến thức OrbitTech**: LLM Judge có thể không hiểu hết các quy định nghiệp vụ ngầm hoặc tầm quan trọng của từng chính sách cụ thể nếu không được căn chỉnh theo đánh giá chuyên môn của con người.
> 2. **Phát hiện và đo lường các thiên vị ẩn (Unintended Biases)**: Giúp phát hiện xem LLM Judge có bị ảnh hưởng bởi Position bias, Verbosity bias, hay Self-preference bias (thích output của chính dòng mô hình đó) hay không.
> 3. **Xác định chỉ số tin cậy (Agreement Rate & Calibration Metrics)**: Giúp tính toán các chỉ số đồng thuận như Cohen's Kappa hoặc Pearson Correlation giữa LLM Judge và Human Annotators. Chỉ khi tỷ lệ đồng thuận đạt mức cao (ví dụ: $\ge 0.80$ hoặc Correlation $\ge 0.85$), ta mới có đủ cơ sở tin cậy để đưa LLM Judge vào tự động hóa trong pipeline CI/CD nhằm thay thế chấm thủ công tốn kém.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | **0.85** | Là chỉ số quan trọng nhất trong CSKH OrbitTech (Zero Hallucination). Trả lời sai thông tin giá, điều kiện bảo hành hay phí hoàn tiền gây hậu quả pháp lý và làm mất uy tín thương hiệu. Nếu rớt dưới 0.85 phải block deployment ngay lập tức. |
| Answer Relevance | **0.80** | Đảm bảo AI trả lời đúng trọng tâm câu hỏi của khách hàng. Ngưỡng 0.80 linh hoạt vừa đủ để cho phép các câu trả lời ngắn gọn hoặc câu hỏi ngược lại để làm rõ (clarification) mà không làm rớt CI/CD pipeline. |
| Completeness | **0.75** | Đảm bảo câu trả lời cung cấp tương đối đầy đủ các thông tin/điều kiện cần thiết. Đặt ngưỡng 0.75 cho phép chấp nhận một số câu trả lời mang tính tóm tắt nhanh nhưng vẫn đáp ứng được nhu cầu cơ bản của người dùng. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> 
> - **Offline Evaluation**:
>   - *Khi nào dùng*: Chạy tự động trong môi trường Development/Staging và trong pipeline CI/CD trước khi merge Pull Request hoặc deploy phiên bản mới.
>   - *Mục đích*: Chạy trên tập Golden Dataset cố định (20 QA) để phát hiện sớm lỗi Regression (sụt giảm hiệu năng), đảm bảo prompt/model mới không làm hỏng các test case cũ mà không ảnh hưởng tới người dùng thật.
> - **Online Evaluation**:
>   - *Khi nào dùng*: Chạy liên tục trên môi trường Production với dữ liệu thoại/chat thực tế từ người dùng (Live Traffic / Telemetry).
>   - *Mục đích*: Giám sát sức khỏe hệ thống theo thời gian thực (Real-time monitoring), phát hiện các câu hỏi mới lạ ngoài Golden Dataset (Out-of-distribution), đo lường tương tác thực tế của người dùng (Thumbs Up/Down, Resolution Rate, Fallback Rate).
> - **Human Review**:
>   - *Khi nào dùng*:
>     1. Xây dựng và thẩm định tập Golden Dataset ban đầu.
>     2. Định kỳ audit ngẫu nhiên 1–5% dữ liệu Production, đặc biệt chú trọng các case LLM Judge cho điểm thấp hoặc người dùng bấm Thumbs Down / yêu cầu gặp tư vấn viên con người.
>     3. Calibrate lại LLM Judge khi thay đổi chính sách kinh doanh hoặc cập nhật Rubric mới.
>   - *Mục đích*: Cung cấp Ground Truth chuẩn xác nhất, phát hiện các điểm mù mà métrics tự động chưa bắt được.

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
