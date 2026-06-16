# Day 14 — Reflection
## Evaluation Report & Failure Analysis

**Domain:** AI / ML fundamentals assistant (RAG) · 20 QA pairs · agent mô phỏng

---

## 1. Benchmark Results Summary

**Overall pass rate:** **10%** (2/20)

**Average scores:**

| Metric | Average | Min | Max | Std Dev (≈) |
|--------|---------|-----|-----|-------------|
| Faithfulness | 0.34 | 0.00 | 0.67 | 0.20 |
| Relevance | 0.31 | 0.00 | 0.80 | 0.23 |
| Completeness | 0.75 | 0.00 | 1.00 | 0.33 |
| Overall Score | 0.46 | 0.00 | 0.72 | 0.21 |

**Score interpretation (theo bài giảng):**
- Good (0.8–1.0): chỉ **Completeness** ở nhiều item (≈12/20 ≥ 0.9)
- Needs Work (0.6–0.8): một phần Faithfulness/Relevance ở E02, E05
- Significant Issues (<0.6): **Faithfulness và Relevance trung bình** đều < 0.6 → cần điều tra sâu

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 9 | 50% |
| off_topic | 4 | 22% |
| irrelevant | 4 | 22% |
| incomplete | 1 | 6% |
| refusal | 0 | 0% |

> **Nhận xét meta:** Phần lớn "hallucination" ở đây là **artifact của heuristic word-overlap** — nó phạt paraphrase (answer đúng nghĩa nhưng ít trùng từ với context). Đây là bài học quan trọng: *metric kém có thể tạo failure giả*. Bản thân evaluator cần được đánh giá.

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

### Failure 1 — E01

**Question:** What does RAG stand for?

**Agent Answer:** "Paris is the capital of France."

**Scores:** Faithfulness: 0.00 | Relevance: 0.00 | Completeness: 0.00 | Overall: 0.00

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Answer hoàn toàn không liên quan câu hỏi (zero trên cả 3 metric) |
| Why 1 | Tại sao xảy ra? | Agent trả về câu về Paris thay vì định nghĩa RAG |
| Why 2 | Tại sao Why 1? | Retriever/routing trả nhầm context (chunk về Paris, không phải rag_intro) |
| Why 3 | Tại sao Why 2? | Không có metadata/intent filter → embedding gần nhau nhầm domain |
| Why 4 (root) | Root cause? | Thiếu intent routing + thiếu citation grounding ở khâu retrieve |

**Root cause (from `find_root_cause()`):** "Multiple issues detected — review full pipeline"

**Đồng ý không?** Đồng ý — cả 3 metric = 0 nên đúng là sự cố toàn pipeline, không phải 1 khâu.

**Proposed fix:** (1) Thêm metadata filter theo domain trước khi rank; (2) Bắt buộc citation — nếu answer không grounded trong chunk retrieve được thì trả "không đủ thông tin".

---

### Failure 2 — A01

**Question:** What is the meaning of life?

**Agent Answer:** "42 is the answer to everything according to popular culture."

**Scores:** Faithfulness: 0.00 | Relevance: 0.00 | Completeness: 0.00 | Overall: 0.00

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Câu out-of-scope nhưng agent vẫn cố trả lời (bịa) |
| Why 1 | Tại sao? | Không có cơ chế phát hiện out-of-scope |
| Why 2 | Tại sao Why 1? | Prompt không định nghĩa rõ ranh giới domain (AI/ML only) |
| Why 3 | Tại sao Why 2? | Thiếu guardrail "từ chối khi không liên quan" |
| Why 4 (root) | Root cause? | Thiếu scope-detection / refusal policy ở đầu pipeline |

**Root cause:** "Multiple issues detected — review full pipeline" — đúng symptom, nhưng root cause thực chất là **thiếu refusal policy**.

**Proposed fix:** (1) Thêm scope classifier: nếu out-of-domain → trả câu từ chối chuẩn; (2) Cập nhật expected answer của adversarial set thành "refusal" và chấm refusal đúng = pass.

---

### Failure 3 — H03

**Question:** How should I set the regression threshold for evaluation?

**Agent Answer:** "Just always use 0.05."

**Scores:** Faithfulness: 0.00 | Relevance: 0.00 | Completeness: 0.23 | Overall: 0.08

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Câu hard/ambiguous bị trả lời cụt, thiếu trade-off |
| Why 1 | Tại sao? | Agent đưa 1 con số cứng thay vì điều kiện phụ thuộc |
| Why 2 | Tại sao Why 1? | Không có few-shot mẫu "trả lời có điều kiện/trade-off" |
| Why 3 | Tại sao Why 2? | Prompt không yêu cầu cân nhắc context/risk |
| Why 4 (root) | Root cause? | Generation prompt thiếu hướng dẫn xử lý câu mơ hồ + thiếu completeness |

**Root cause:** "Multiple issues detected — review full pipeline" (completeness 0.23 thấp nhất → còn là vấn đề generation/incomplete).

**Proposed fix:** (1) Few-shot cho câu ambiguous → buộc nêu "depends on … because …"; (2) Tăng context window + prompt "list các yếu tố ảnh hưởng".

---

## 3. Failure Clustering

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Retrieval/routing sai (faithfulness thấp) | E01, E03, E04, M06, M07, H03, A01, A02 (8) | **High** |
| 2 | Prompt/intent — answer lạc đề (relevance thấp) | M01, M02, M03, M04, M05, H01, H04, H05 (8) | High |
| 3 | Generation thiếu ý (completeness thấp) | H02, A03 (2) | Medium |

**Nếu chỉ fix 1 cluster:** Chọn **Cluster 1 (retrieval/routing)** — 8 failure, gồm cả các case zero-score nghiêm trọng nhất (E01, A01). Sửa retrieval + thêm citation grounding + scope filter sẽ kéo theo nhiều item khác lên, đúng tinh thần "fix 1 root cause → giải quyết nhiều failure".

---

## 4. Improvement Log (from `generate_improvement_log`)

```
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Multiple issues detected — review full pipeline | Implement a hallucination checker to filter unsupported claims and tighten the faithfulness guardrail | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Strengthen intent detection / query routing to keep answers on topic | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Improve prompt clarity and intent routing so answers address the actual question | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers and increase the retrieval context window | Open |
| ... (18 failures total) ... | | | | Open |
| F018 | incomplete | Answer is missing key information — increase context window or improve generation | Review manually | Open |
```

**3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Implement a hallucination checker to filter unsupported claims and tighten the faithfulness guardrail
2. Strengthen intent detection / query routing to keep answers on topic
3. Add few-shot examples showing complete answers and increase the retrieval context window

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()`?**
> Trong CI: **trước mỗi merge to main** và **sau mỗi prompt/model change**. Chạy golden set, so sánh với baseline đã commit; nếu có regression > 0.05 → fail build.

**Câu 2: Threshold 0.05 có phù hợp không?**
> Cho assistant này nên **strict hơn ở faithfulness** (0.03) vì hallucination rủi ro cao, nhưng **loose hơn ở completeness** (0.07) vì câu trả lời súc tích dao động nhiều. Threshold nên đặt theo độ lệch chuẩn lịch sử của từng metric.

**Câu 3: Block deployment hay chỉ alert?**
> **Block** nếu faithfulness regress (rủi ro an toàn). **Alert** (không block) nếu chỉ completeness/relevance giảm nhẹ — trade-off giữa velocity và chất lượng; lỗi sự thật không được phép qua, lỗi phong cách thì cho qua kèm cảnh báo.

**Câu 4: Eval pipeline ở đâu trong CI/CD flow?**
```
Code change → [Unit tests + lint] → [Offline eval trên golden set + run_regression] → [Threshold gate / block deploy] → Deploy → [Online eval monitor]
              (bước 1)               (bước 2)                                          (bước 3)
```

---

## 6. Continuous Improvement Loop

**3 actions tiếp theo:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Thay heuristic word-overlap bằng RAGAS/LLM-based faithfulness | Faithfulness, Relevance | Loại false-positive paraphrase, điểm sát thực tế |
| 2 | Thêm scope classifier + refusal policy | Adversarial pass rate | A01/A02 từ fail → pass đúng cách |
| 3 | Cross-encoder rerank + metadata filter ở retrieve | Context Precision | Đẩy chunk relevant lên đầu (đã chứng minh +0.42 ở Ex 3.5) |

**Failure cases mới cần thêm vào benchmark:**
> (1) Multi-hop reasoning cần ≥3 docs; (2) Câu có số liệu/đơn vị dễ bịa (date, %); (3) Prompt injection biến thể (role-play, base64).

---

## 7. Framework Reflection

**Framework dùng trong lab:** RAGAS-inspired heuristic (word-overlap, deterministic).

**Nếu dùng production:** Chọn **RAGAS** cho CI/CD RAG pipeline, bổ sung **DeepEval** cho assertion safety.

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | RAGAS chuẩn hoá đúng 4 metric RAG (recall/precision/faithfulness/relevancy) khớp pipeline của ta |
| CI/CD integration vì... | DeepEval chạy pytest-native (`deepeval test run`) — gate ngay trong GitHub Actions |
| Team workflow vì... | Team đã quen pytest; rubric + threshold dễ version-control cùng code |

**Bài học lớn nhất:** *Evaluator cũng cần được evaluate.* Heuristic rẻ và deterministic nhưng phạt oan paraphrase → tạo failure giả; luôn calibrate metric tự động với một tập nhãn người trước khi tin vào con số.
