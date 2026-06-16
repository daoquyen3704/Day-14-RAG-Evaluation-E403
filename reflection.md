# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Kết quả từ Exercise 3.2:

**Overall pass rate:** 5.0%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.29 | 0.00 | 0.71 | 0.19 |
| Relevance | 0.33 | 0.00 | 0.80 | 0.19 |
| Completeness | 0.68 | 0.12 | 1.00 | 0.33 |
| Overall Score | 0.43 | 0.17 | 0.67 | 0.15 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8–1.0)? 0
- Bao nhiêu metrics ở Needs Work (0.6–0.8)? 1 (Completeness là 0.68)
- Bao nhiêu metrics ở Significant Issues (<0.6)? 3 (Faithfulness, Relevance, Overall Score)

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 11 | 55.0% |
| irrelevant | 4 | 20.0% |
| incomplete | 1 | 5.0% |
| off_topic | 3 | 15.0% |
| refusal | 0 | 0.0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

### Failure 1

**Question:** *How does query expansion/rewriting address user queries with poor phrasing?*

**Agent Answer:** *Query expansion is when you expand the size of your query by adding useless letters like AAAAAAA to make it longer.*

**Scores:** Faithfulness: 0.14 | Relevance: 0.20 | Completeness: 0.17 | Overall: 0.17

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Agent trả lời hoàn toàn sai lệch và vô nghĩa về query expansion. |
| Why 1 | Tại sao xảy ra? | Vì agent không có kiến thức đúng đắn và tự bịa ra định nghĩa vô nghĩa ("adding useless letters like AAAAAAA"). |
| Why 2 | Tại sao Why 1 xảy ra? | Vì prompt không yêu cầu agent sử dụng context và không có quy tắc nghiêm ngặt bắt buộc trả lời dựa vào tài liệu. |
| Why 3 | Tại sao Why 2 xảy ra? | Vì context retrieve được về query expansion không được ưu tiên đưa vào prompt hoặc bị chôn vùi trong đống tài liệu gây nhiễu. |
| Why 4 | Root cause là gì? | Thiếu bộ lọc tính trung thực (hallucination guardrail) ở đầu ra và retriever không sắp xếp tài liệu tối ưu (Context Precision thấp). |

**Root cause (from `find_root_cause()`):**
> Context is missing or irrelevant — improve retrieval

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Có. Vì retriever lấy thông tin không chính xác hoặc agent không bám sát context, dẫn đến hallucination.

**Proposed fix (cụ thể, actionable):**
> - Tăng cường hệ thống Reranking để xếp các chunk có chứa thông tin chính xác về query expansion lên đầu.
> - Thêm bộ lọc tính trung thực (hallucination filter) trước khi xuất kết quả.

---

### Failure 2

**Question:** *Should I use RAG or fine-tuning to update my chatbot's knowledge with new proprietary data?*

**Agent Answer:** *You should use RAG for chatbots.*

**Scores:** Faithfulness: 0.20 | Relevance: 0.21 | Completeness: 0.12 | Overall: 0.18

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Câu trả lời quá ngắn, không so sánh được giữa RAG và fine-tuning. |
| Why 1 | Tại sao xảy ra? | Vì agent chỉ đưa ra kết luận cực ngắn "use RAG" mà bỏ qua mọi thông tin giải thích, so sánh. |
| Why 2 | Tại sao Why 1 xảy ra? | Vì generator không được hướng dẫn phải viết câu trả lời đầy đủ, so sánh chi tiết các mặt lợi hại trong prompt. |
| Why 3 | Tại sao Why 2 xảy ra? | Vì bộ dữ liệu và prompt thiếu các few-shot examples hướng dẫn cách trả lời so sánh/đánh giá. |
| Why 4 | Root cause là gì? | Generator có Completeness thấp, thiếu hướng dẫn chi tiết trong prompt và thiếu few-shot training. |

**Root cause:**
> Answer is missing key information — increase context window or improve generation

**Proposed fix:**
> - Thêm few-shot examples dạng so sánh trong prompt.
> - Bổ sung chỉ dẫn "Hãy so sánh chi tiết ưu nhược điểm của cả 2 giải pháp dựa trên chi phí, độ cập nhật và thời gian huấn luyện".

---

### Failure 3

**Question:** *What is hallucination in LLMs and how does RAG help reduce it?*

**Agent Answer:** *Hallucination is when a model sings songs, and RAG helps by playing music.*

**Scores:** Faithfulness: 0.22 | Relevance: 0.25 | Completeness: 0.20 | Overall: 0.22

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Trả lời sai thực tế, ví von vô nghĩa ("model sings songs", "RAG plays music"). |
| Why 1 | Tại sao xảy ra? | Vì agent bịa thông tin (hallucination) thay vì dùng thông tin kỹ thuật trong context. |
| Why 2 | Tại sao Why 1 xảy ra? | Vì prompt hệ thống lỏng lẻo, không phạt hoặc ngăn chặn việc tự sáng tạo thông tin ngoài context. |
| Why 3 | Tại sao Why 2 xảy ra? | Vì không có cơ chế tự kiểm tra chéo (self-consistency check) hoặc chấm điểm trung thực trước khi trả về cho user. |
| Why 4 | Root cause là gì? | Faithfulness cực kỳ thấp do thiếu ràng buộc prompt chặt chẽ và thiếu hallucination guardrail. |

**Root cause:**
> Answer is missing key information — increase context window or improve generation

**Proposed fix:**
> - Sử dụng prompt ép buộc: "Nếu không tìm thấy thông tin trong context, hãy trả lời 'Tôi không biết'."
> - Thêm bộ lọc trung thực (faithfulness filter) trước khi xuất kết quả.

---

## 3. Failure Clustering

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Hallucination do prompt lỏng lẻo và thiếu guardrail. | 11 | High |
| 2 | Irrelevant / Off-topic do intent classification kém và prompt không rõ ràng. | 7 | Medium |
| 3 | Incomplete do thiếu dữ liệu ngữ cảnh hoặc giới hạn câu trả lời quá ngắn. | 1 | Medium |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn **Cluster 1 (Hallucination)**. Vì hallucination là lỗi nghiêm trọng nhất trong các hệ thống RAG thực tế, ảnh hưởng trực tiếp đến độ tin cậy và an toàn thông tin của chatbot. Khắc phục được lỗi này sẽ cải thiện đáng kể điểm Faithfulness và trải nghiệm người dùng.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F003 | off_topic | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | irrelevant | Answer does not address the question — improve prompt clarity | Add system prompt instructions to focus strictly on the query context | Open |
| F005 | irrelevant | Answer does not address the question — improve prompt clarity | Implement intent detection to route off-topic queries to a default response handler | Open |
| F006 | irrelevant | Answer does not address the question — improve prompt clarity | Tune prompt to penalize long-winded irrelevant explanations | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F008 | off_topic | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F009 | hallucination | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F011 | incomplete | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F012 | hallucination | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F013 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F017 | hallucination | Multiple issues detected — review full pipeline | N/A | Open |
| F018 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F019 | off_topic | Multiple issues detected — review full pipeline | N/A | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Implement hallucination checker to filter unsupported claims
2. Increase chunk size in RAG pipeline to reduce context fragmentation
3. Add few-shot examples showing complete answers to improve completeness

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Chạy `run_regression()` trước mỗi lần merge Pull Request vào nhánh `main` (CI gate), sau mỗi khi chỉnh sửa Prompt hệ thống, nâng cấp phiên bản mô hình LLM, hoặc thay đổi cấu hình Retriever (như chunk size, thuật toán embedding).

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> Khá phù hợp cho giai đoạn thử nghiệm nhưng đối với hệ thống sản xuất (production) có yêu cầu cao về độ chính xác, ngưỡng này nên được thắt chặt hơn (ví dụ drop > 0.02) để kịp thời phát hiện các suy giảm chất lượng nhỏ nhưng tích tụ.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> Nên **block deployment** nếu sự sụt giảm xảy ra ở metric trọng yếu như Faithfulness (giảm > 0.05) vì có nguy cơ cao chatbot phát tán thông tin sai lệch. Đối với các metric khác như Relevance hoặc Completeness, có thể cấu hình **alert** để team xem xét thủ công trước khi quyết định deploy, nhằm tránh nghẽn pipeline CI/CD khi chỉ bị suy giảm nhẹ.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```
Code change → [Unit Tests] → [RAG Eval (Benchmark Run)] → [Regression Check] → Deploy
              (bước 1)       (bước 2)                     (bước 3)
```
> *Điền 3 bước eval vào flow trên:*
> - Bước 1: Unit Tests (pytest cho codebase)
> - Bước 2: RAG Eval (chạy benchmark trên Golden Dataset)
> - Bước 3: Regression Check (đối chiếu điểm số trung bình với baseline)

---

## 6. Continuous Improvement Loop

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Tích hợp Reranker (như BGE-Reranker) | Context Precision | Tăng độ chính xác của ngữ cảnh đầu vào, đẩy thông tin đúng lên đầu. |
| 2 | Cải tiến Prompt hệ thống với Few-shot Examples | Completeness | Cải thiện độ phủ thông tin và định hướng câu trả lời đúng định dạng. |
| 3 | Triển khai Hallucination Filter | Faithfulness | Loại bỏ các câu trả lời bịa đặt trước khi gửi tới người dùng. |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> - Các câu hỏi chứa thông tin lỗi thời (tập trung test khả năng cập nhật của retriever).
> - Các câu hỏi có từ khóa trùng lặp nhưng ngữ nghĩa khác nhau (test độ phân giải ngữ nghĩa của vector search).
> - Các câu hỏi dài dòng của người dùng thực tế từ logs để test tính bền bỉ của prompt.

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** RAGAS-inspired heuristic (word-overlap)

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | RAGAS cung cấp các metric đánh giá RAG chuyên sâu được chuẩn hóa quốc tế và liên tục cập nhật thuật toán đánh giá bằng LLM-as-Judge cao cấp. |
| CI/CD integration vì... | DeepEval hỗ trợ pytest native rất mạnh mẽ, giúp tích hợp trực tiếp vào GitHub Actions cực kỳ đơn giản với các assert câu lệnh ngắn. |
| Team workflow vì... | Cả hai đều hỗ trợ lưu trữ logs đánh giá trực tuyến (dashboard) giúp cả team dễ dàng theo dõi chất lượng qua từng sprint. |
