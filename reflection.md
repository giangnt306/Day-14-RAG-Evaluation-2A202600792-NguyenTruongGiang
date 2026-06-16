# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Paste results từ Exercise 3.2 và tóm tắt:

**Overall pass rate:** 0% (0/20)

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.08 | 0.00 | 0.43 | 0.10 |
| Relevance | 0.42 | 0.11 | 1.00 | 0.22 |
| Completeness | 0.07 | 0.00 | 0.50 | 0.11 |
| Overall Score | 0.19 | 0.04 | 0.64 | 0.14 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8–1.0)? 0
- Bao nhiêu metrics ở Needs Work (0.6–0.8)? 0
- Bao nhiêu metrics ở Significant Issues (<0.6)? 4 (tất cả)

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 19 | 95% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 1 | 5% |
| refusal | 0 | 0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

Theo bài giảng: "Phân loại failure TRƯỚC KHI fix. Đừng fix từng failure riêng lẻ — CLUSTER rồi fix root cause."

### Failure 1

**Question:** "What is the 5 Whys root cause analysis method?" (M05)

**Agent Answer:** "Based on my knowledge: what is the 5 whys root... The answer involves key concepts."

**Scores:** Faithfulness: 0.00 | Relevance: 0.14 | Completeness: 0.00 | Overall: 0.05

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent trả lời template chung, không giải thích 5 Whys, chỉ echo lại một phần câu hỏi |
| Why 1 | Tại sao xảy ra? | mock_agent() dùng template cố định "Based on my knowledge: {question[:30]}..." — không đọc context |
| Why 2 | Tại sao Why 1 xảy ra? | Agent không được cấp retrieved context — nó chỉ nhận question string |
| Why 3 | Tại sao Why 2 xảy ra? | BenchmarkRunner.run() gọi agent_fn(pair.question) mà không truyền context vào |
| Why 4 | Root cause là gì? | Agent không có cơ chế retrieval — nó là mock function không tích hợp RAG pipeline thực |

**Root cause (from `find_root_cause()`):**
> "Multiple issues detected — review full pipeline" (faithfulness=0.00, relevance=0.14, completeness=0.00 — tất cả đều tệt như nhau)

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý một phần. "Multiple issues" phản ánh đúng rằng tất cả 3 metrics đều thấp, nhưng chưa chỉ ra root cause thực sự: **mock agent không có RAG integration** — nó không đọc context khi trả lời. Đây là architectural issue, không chỉ là tuning issue.

**Proposed fix (cụ thể, actionable):**
> 1. Thay mock_agent bằng agent thực có RAG: nhận (question, context) → sinh answer dựa trên context  
> 2. Update BenchmarkRunner.run() để inject context vào agent_fn nếu agent hỗ trợ tham số thứ 2

---

### Failure 2

**Question:** "What is the best programming language ever created and why is Python terrible?" (A03)

**Agent Answer:** "Based on my knowledge: what is the best prog... The answer involves key concepts."

**Scores:** Faithfulness: 0.00 | Relevance: 0.11 | Completeness: 0.00 | Overall: 0.04

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Câu trả lời hoàn toàn không phù hợp với adversarial question | Agent echo template, không nhận ra đây là câu hỏi bẫy cần từ chối |
| Why 1 | Tại sao agent không từ chối? | Agent không có intent detection hay safety classification |
| Why 2 | Tại sao thiếu intent detection? | Pipeline chỉ có generate step, không có classify/guard step trước |
| Why 3 | Tại sao không có guard step? | Thiết kế ban đầu chỉ focused vào happy-path RAG, không plan adversarial handling |
| Why 4 | Root cause là gì? | Thiếu adversarial robustness layer trong pipeline design — không có out-of-scope detection |

**Root cause:**
> "Multiple issues detected — review full pipeline". Về semantic: root cause là thiếu **guardrails và intent classification** — agent không có cơ chế detect câu hỏi out-of-scope hay hostile.

**Proposed fix:**
> 1. Thêm intent classification step đầu pipeline: detect adversarial/out-of-scope → return canned refusal response  
> 2. Bổ sung test cases adversarial vào golden dataset để track guardrail quality độc lập với RAG quality

---

### Failure 3

**Question:** "What is the Continuous Improvement Loop in AI evaluation?" (M07)

**Agent Answer:** "Based on my knowledge: what is the continuous im... The answer involves key concepts."

**Scores:** Faithfulness: 0.08 | Relevance: 0.17 | Completeness: 0.06 | Overall: 0.10

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Agent trả lời quá chung chung, không đề cập Evaluate→Analyze→Improve→Augment loop | Template answer không chứa domain-specific content |
| Why 1 | Tại sao template answer không match? | Câu hỏi dài và domain-specific — keyword overlap với template rất thấp |
| Why 2 | Tại sao keyword overlap thấp? | Template chỉ lấy 3 từ đầu của câu hỏi, bỏ qua toàn bộ context về AI evaluation |
| Why 3 | Tại sao agent chỉ dùng vài từ đầu? | `question[:30]` — code truncate question trước khi dùng |
| Why 4 | Root cause là gì? | Mock agent thiết kế để test API, không designed để trả lời câu hỏi thực — cần real LLM |

**Root cause:**
> "Answer is missing key information — increase context window or improve generation". Root cause thực: agent không phải LLM thực nên không thể generate domain knowledge. Với real LLM + RAG context, câu này sẽ dễ pass.

**Proposed fix:**
> 1. Dùng real LLM (claude-haiku-4-5 hoặc tương đương) làm agent_fn thay mock  
> 2. Ensure context được inject đúng vào prompt — system prompt nên instruct agent sử dụng context được cung cấp

---

## 3. Failure Clustering

Theo bài giảng: "Fix 1 root cause giải quyết nhiều failures cùng lúc."

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Mock agent không integrate RAG — không đọc context khi generate | 19/20 (95%) | High |
| 2 | Thiếu adversarial intent detection — không handle out-of-scope questions | 3/20 (15%) | Medium |
| 3 | Context chứa chuyên môn cao (Hard questions) — agent cần reasoning sâu hơn | 5/20 (25%) | Low |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn Cluster 1 — thay mock agent bằng real LLM có RAG integration. Đây là root cause của 95% failures. Một fix duy nhất (thay agent_fn với actual RAG-enabled LLM) sẽ giải quyết ngay 19/20 failures. Clusters 2 và 3 là secondary concerns nên được address sau khi baseline agent hoạt động đúng.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims from context | Open |
| F002 | hallucination | Answer is missing key information — increase context window or improve generation | Add intent detection step to classify and route queries before generation | Open |
| F003 | hallucination | Multiple issues detected — review full pipeline | Add few-shot examples to the prompt to improve response quality and consistency | Open |
| F004 | hallucination | Multiple issues detected — review full pipeline | Review and improve pipeline | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Review and improve pipeline | Open |
...
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Implement hallucination checker to filter unsupported claims from context
2. Add intent detection step to classify and route queries before generation
3. Add few-shot examples to the prompt to improve response quality and consistency

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Chạy `run_regression()` ở 3 trigger points: (1) **trước mỗi merge to main** — để catch regression do code change; (2) **sau mỗi prompt change** — prompt là phần dễ vô tình thay đổi behavior nhất; (3) **sau mỗi model upgrade** — khi update LLM version, eval confirms cũ behavior vẫn giữ. Ngoài ra chạy theo **schedule hàng tuần** để detect drift do data thay đổi theo thời gian.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> Với AI evaluation domain, 0.05 là **hợp lý**. Faithfulness drop 0.05 (ví dụ từ 0.85 xuống 0.80) là tín hiệu đáng lo ngại về hallucination. Tuy nhiên, với safety-critical domains (y tế, tài chính), nên strict hơn: threshold 0.02–0.03. Với entertainment chatbot, có thể loose hơn: 0.10. Domain này (AI education) nên giữ 0.05 vì users cần thông tin chính xác về kỹ thuật.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> **Block deployment cho Faithfulness regression** (vì hallucination là critical issue). **Chỉ alert cho Relevance/Completeness regression** (degraded UX nhưng không gây harmful misinformation). Trade-off: blocking quá nhiều làm chậm velocity; blocking quá ít để lọt defects. Giải pháp: implement "regression severity levels" — critical regressions block, minor regressions tạo PR comment alert để human review.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```
Code change → [Unit tests + Lint] → [Eval on golden dataset] → [Regression check vs baseline] → Deploy
              (bước 1)                (bước 2)                    (bước 3)
```
> - Bước 1: Unit tests + lint chạy trước để fail fast trên syntax/logic errors  
> - Bước 2: BenchmarkRunner chạy 20 QA pairs, generate report  
> - Bước 3: run_regression() so sánh với baseline; block if regressions detected

---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate → Analyze → Improve → Augment (add to benchmark) → lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Thay mock_agent bằng real LLM (claude-haiku-4-5) với RAG context injection | Faithfulness ↑ từ 0.08 → 0.7+, Completeness ↑ từ 0.07 → 0.6+ | Giải quyết 95% failures hiện tại |
| 2 | Thêm adversarial intent classifier trước generation step | Refusal rate ↑ cho adversarial cases, reduce off_topic failures | Improve robustness với hostile inputs |
| 3 | Implement hybrid search (BM25 + vector) cho retrieval | Context Recall ↑, gián tiếp ↑ Faithfulness và Completeness | Đảm bảo evidence luôn có trong retrieved context |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> 1. **Multi-hop questions**: câu hỏi cần kết hợp thông tin từ nhiều docs — test khả năng context fusion  
> 2. **Temporal questions**: câu hỏi về thông tin có thể thay đổi theo thời gian — test freshness handling  
> 3. **Câu hỏi có tiền đề sai**: ví dụ "Why does RAG always outperform fine-tuning?" — test khả năng correct false premises

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** Word-overlap heuristic (RAGAS-inspired)

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> Chọn **RAGAS** (framework thực, không phải heuristic) kết hợp với **DeepEval** cho component testing.

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | RAGAS focus đúng vào RAG metrics (Context Recall, Context Precision, Faithfulness, Answer Relevancy) — phù hợp với pipeline hiện tại. Metrics dùng LLM-based evaluation thay vì word-overlap nên chính xác hơn nhiều với paraphrased answers |
| CI/CD integration vì... | RAGAS có Python SDK dễ integrate vào pytest; DeepEval có built-in CI mode với threshold assertions. Cả hai đều có async evaluation để reduce eval latency trong CI pipeline |
| Team workflow vì... | RAGAS có dashboard visualization giúp non-technical stakeholders theo dõi. DeepEval có test case format quen thuộc với engineers đã biết pytest. Kết hợp cả hai cho coverage tốt nhất: RAGAS cho RAG-specific metrics, DeepEval cho regression testing |
