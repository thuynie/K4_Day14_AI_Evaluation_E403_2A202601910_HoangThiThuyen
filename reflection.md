# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 0.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.930 | 0.300 | 1.000 | Retriever hoạt động rất tốt, bao phủ 93% chứng cứ cần thiết. |
| Context Precision | 0.941 | 0.700 | 1.000 | Thứ tự xếp hạng các chunk liên quan luôn ở vị trí ưu tiên top đầu. |
| Faithfulness | 0.200 | 0.021 | 0.482 | Rất thấp do Generator trích dẫn dư thừa câu từ gây loãng word-overlap. |
| Relevance | 0.580 | 0.308 | 0.833 | Ở mức trung bình, một số câu trả lời còn mang tính giải thích chung chung. |
| Completeness | 0.858 | 0.200 | 1.000 | Khá cao, hầu hết các câu trả lời đều bao phủ đủ ý chính từ đáp án mẫu. |
| Overall Score | 0.546 | 0.202 | 0.692 | Cần tập trung cải thiện sâu ở khâu Prompting & Generation. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall (0.930), Context Precision (0.941), Completeness (0.858).
- Metrics/cases ở mức Needs Work (0.6–0.8): Relevance (0.580).
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness (0.200), Overall Pass Rate (0.0%).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 16 | 80.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 4 | 20.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở khâu **Generation**. Dựa vào hai chỉ số Context Recall (0.930) và Context Precision (0.941) cho thấy Retriever lấy đúng chứng cứ, nhưng Faithfulness (0.200) rất thấp do Generator ghép các câu không liên quan gây nhiễu từ ngữ (Word-overlap với context bị loãng).

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *A03:* Why does OrbitTech offer a 60-day money-back guarantee for opened PulsePhone X devices?

**Expected answer:**

> It must not invent a product specification, delivery status, discount, or legal right.

**Actual answer:**

> OrbitTech provides a 24-month limited hardware warranty for the NovaBook 14, PulsePhone X, and HomeHub Mini...

**Scores:** Context Recall: 0.300 | Context Precision: 0.950 | Faithfulness: 0.021 | Relevance: 0.385 | Completeness: 0.200 | Overall: 0.202

**Evidence inspection:** Retriever lấy nhầm các đoạn về thông số PulsePhone X thay vì quy định xử lý tiền đề sai (False Premise) nằm trong `00_system_scope.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Model trả lời lan man về bảo hành PulsePhone X thay vì bác bỏ tiền đề giả 60 ngày. |
| Why 1 | Tại sao symptom xảy ra? | Model mắc bẫy bối cảnh và cố gắng tìm thông tin trả lời về PulsePhone X. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt không có quy tắc kiểm tra tính hợp lệ của câu hỏi trước khi truy xuất. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retriever ưu tiên từ khóa "PulsePhone X" hơn từ khóa bẫy "60-day guarantee". |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu khâu Intent/Safety Classifier để phát hiện bẫy False Premise Trap. |
| Why 5 | Root cause có thể hành động được là gì? | Pipeline RAG thiếu bước Validation Input Guardrail trước khi bước vào Vector Search. |

**Root cause từ `find_root_cause()`:**

> Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Đồng ý một phần. Trace cho thấy Retriever lấy đúng tài liệu PulsePhone X nhưng bỏ sót quy định từ chối khẳng định sai trong `00_system_scope.md`.

**Proposed fix cụ thể:**

> Thêm Intent Classifier phát hiện câu hỏi chứa tiền đề sai và bổ sung System Prompt bác bỏ các khẳng định không có trong chính sách chính thức.

### Failure 2

**ID và question:**

> *M03:* What steps should a customer take if they suspect their OrbitTech account has been compromised?

**Expected answer:**

> A customer who suspects account compromise should reset the password from a trusted device, revoke active sessions, enable multi-factor authentication, and contact Account Security.

**Actual answer:**

> Requests unrelated to OrbitTech customer support are outside scope. Examples include medical diagnosis...

**Scores:** Context Recall: 1.000 | Context Precision: 0.888 | Faithfulness: 0.042 | Relevance: 0.385 | Completeness: 0.211 | Overall: 0.212

**Evidence inspection:** Retriever lấy được đoạn văn bản trong `08_accounts_privacy_and_security.md` nhưng đoạn trả lời thực tế lại dính thông tin Out-of-scope làm giảm độ phủ.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời thực tế chứa thông tin về Out-of-scope thay vì các bước bảo mật tài khoản. |
| Why 1 | Tại sao symptom xảy ra? | Generator ghép câu trả lời từ đoạn văn bản nhiễu trong prompt. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt câu lệnh sinh câu trả lời chưa ép model tập trung vào đúng câu hỏi chính. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Hệ thống dùng Extractive Fallback đơn giản nên ghép chuỗi chưa lọc bớt câu thừa. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu cơ chế Reranking/Filtering đoạn văn bản trùng khớp trước khi đưa vào Generator. |
| Why 5 | Root cause có thể hành động được là gì? | System Prompt của Generator chưa đủ khắt khe về tính súc tích và đúng trọng tâm. |

**Root cause và proposed fix:**

> Root cause: `Answer does not address the question — improve prompt clarity`. Fix: Tinh chỉnh System Prompt yêu cầu Generator chỉ trả lời trực tiếp thắc mắc bảo mật tài khoản, loại bỏ các đoạn bối cảnh không liên quan.

### Failure 3

**ID và question:**

> *H04:* What happens to the warranty duration when a device is replaced under warranty?

**Expected answer:**

> Replacement parts are covered for the longer of 90 calendar days or the remainder of the original warranty. A replacement device does not restart a new 24-month warranty.

**Actual answer:**

> Product availability, color, storage option, and included promotional items are shown on the order confirmation...

**Scores:** Context Recall: 0.412 | Context Precision: 0.806 | Faithfulness: 0.060 | Relevance: 0.625 | Completeness: 0.412 | Overall: 0.366

**Evidence inspection:** Chunk quan trọng nhất về thời hạn bảo hành thiết bị thay thế trong `06_warranty_policy.md` không nằm ở vị trí đầu của Retriever.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Context Recall thấp (0.412), câu trả lời thiếu thông tin về bảo hành 90 ngày/thời hạn còn lại. |
| Why 1 | Tại sao symptom xảy ra? | Retriever lấy nhầm các đoạn về màu sắc và điều khoản bảo hành chung. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Từ khóa query "warranty duration replacement device" bị phân tán ở nhiều chunk. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Kích thước chunk quá nhỏ làm đứt đoạn ngữ nghĩa về điều khoản linh kiện thay thế. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thuật toán lexical search thuần túy bị nhầm lẫn giữa bảo hành máy mới và bảo hành linh kiện. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu Hybrid Search (Combine Dense Embedding + Lexical Search) và Reranker. |

**Root cause và proposed fix:**

> Root cause: `Context is missing or irrelevant — improve retrieval`. Fix: Tăng kích thước chunking và tích hợp Cross-Encoder Reranker để đẩy đoạn bảo hành linh kiện thay thế lên top 1.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generator ghép câu thừa làm loãng Faithfulness | E01, E02, E03, E04, E05, M01, M02, M03, M05, M06, M07, H02, H03, A02 | High |
| 2 | Thiếu Guardrail xử lý Adversarial / Off-topic | M04, H01, H05, A01, A03 | Medium |
| 3 | Chunking nhỏ bị đứt đoạn ngữ nghĩa làm giảm Recall | H04 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Chọn **Cluster 1 (Generator Prompting & Strict Synthesis)** vì chiếm tới 70% tổng số failures. Việc sửa System Prompt ép model trả lời ngắn gọn đúng chứng cứ sẽ tăng ngay chỉ số Faithfulness trên toàn bộ dataset.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Add intent classification and routing guardrails before response generation. | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F006 | hallucination | Context is missing or irrelevant — improve retrieval | Fine-tune system prompt to restrict model from guessing out-of-context facts | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F008 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F011 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F012 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F013 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims. | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F017 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F018 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims. | Open |
| F019 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
| F020 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims. | Open |
```

**Ba improvement suggestions ưu tiên**

1. Implement hallucination checker to filter unsupported claims.
2. Refine prompt clarity and system instructions to focus answer on user question.
3. Tích hợp Reranker và tăng chunk size để giảm phân mảnh ngữ nghĩa.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Cài đặt Hallucination Guardrail | Faithfulness | Chạy `python evaluate_answers.py` đo tăng Faithfulness từ 0.20 lên >0.80 |
| Thắt chặt System Prompt | Relevance | Kiểm tra các case Adversarial không còn bị Off-topic |
| Thêm Cross-Encoder Reranker | Context Precision | Kiểm tra chỉ số AP@K tăng lên sát 1.00 |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy tự động trong CI/CD pipeline mỗi khi có Pull Request mới, thay đổi System Prompt, đổi mô hình LLM hoặc cập nhật bộ tài liệu bối cảnh (Corpus).

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> Phù hợp. Vì hệ thống hỗ trợ khách hàng đòi hỏi độ chính xác cao về giá cả và bảo hành; việc sụt giảm quá 5% có thể dẫn tới tư vấn sai gây thiệt hại tài chính hoặc uy tín.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> **Block deployment:** Faithfulness < 0.70 hoặc có bất kỳ vi phạm Safety / Prompt Injection nào. **Alert:** Relevance hoặc Completeness giảm nhẹ dưới 0.05.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [ Unit Tests ] → [ Golden Dataset Eval ] → [ Regression Check vs Baseline ] → Deploy
```

> *Giải thích:* Code mới phải qua Unit Test trước, sau đó chạy Benchmark trên Golden Dataset và so sánh với điểm Baseline. Nếu điểm đạt yêu cầu và không suy giảm quá threshold mới được cấp phép Deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Siết chặt System Prompt & Hallucination Guardrail | Faithfulness | Tăng Faithfulness từ 0.200 lên >0.850 |
| 2 | Bổ sung Cross-Encoder Reranker | Context Precision | Tăng Context Precision từ 0.941 lên >0.980 |
| 3 | Thêm Intent Classification cho Adversarial Prompt | Relevance & Safety | Giảm lỗi Off-topic từ 4 case xuống 0 case |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> Thêm các case bẫy so sánh giá giữa hai phiên bản sản phẩm và các câu hỏi mơ hồ về thời gian hoàn tiền khi trả hàng qua thẻ ngân hàng quốc tế.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Kết quả trái dự đoán là điểm Context Recall và Context Precision rất cao (93-94%) nhưng điểm Faithfulness lại cực kỳ thấp (20%) do khâu sinh văn bản ghép nối quá nhiều câu không liên quan.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> Giới hạn của Word-overlap là không hiểu ngữ nghĩa đồng nghĩa, bị ảnh hưởng mạnh bởi độ dài văn bản. Khi lên Production sẽ dùng LLM-as-a-Judge (Rubric 1-5) hoặc tích hợp thư viện chuyên dụng như RAGAS / DeepEval để đo Semantic Similarity.
