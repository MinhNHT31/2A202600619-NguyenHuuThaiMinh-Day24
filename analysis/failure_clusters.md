# Failure Cluster Analysis — Phase A

**Sinh viên:** Nguyễn Hữu Thái Minh  
**Ngày:** 30/06/2026

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.9208 | 0.5892 | 0.8000 |
| answer_relevancy | 0.1843 | 0.1641 | 0.3697 |
| context_precision | 0.9375 | 0.9417 | 0.9667 |
| context_recall | 0.9000 | 0.7792 | 0.6500 |
| **avg_score** | **0.7356** | **0.6185** | **0.6966** |

---

## 2. Bottom 10 Questions

| Rank | Distribution | Question | avg_score | worst_metric |
|---|---|---|---|---|
| 1 | multi_hop | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | 0.0000 | faithfulness |
| 2 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.3333 | faithfulness |
| 3 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.3750 | faithfulness |
| 4 | adversarial | Bao lâu phải đổi mật khẩu một lần? | 0.3955 | faithfulness |
| 5 | factual | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | 0.5000 | faithfulness |
| 6 | multi_hop | Nhân viên đi công tác nước ngoài 4 ngày tại thành phố lớn, ở khách sạn 200 USD/đêm (3 đêm). Công ty thanh toán bao nhiêu tiền khách sạn? | 0.5157 | faithfulness |
| 7 | multi_hop | Nhân viên đi công tác trong nước 2 ngày, ở khách sạn giá 1.500.000 VNĐ/đêm. Công ty thanh toán tối đa bao nhiêu cho tiền khách sạn? | 0.5168 | answer_relevancy |
| 8 | multi_hop | Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu? | 0.5321 | answer_relevancy |
| 9 | multi_hop | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT? | 0.5814 | answer_relevancy |
| 10 | factual | Nhân viên chính thức được phép làm việc từ xa tối đa bao nhiêu ngày một tuần? | 0.5907 | answer_relevancy |

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 2 | 5 | 2 | 9 |
| answer_relevancy | 18 | 15 | 5 | 38 |
| context_precision | 0 | 0 | 0 | 0 |
| context_recall | 0 | 0 | 3 | 3 |

---

## 4. Dominant Failure Analysis

**Dominant distribution:** factual  
**Dominant metric:** answer_relevancy

**Lý do phân tích:**

Điểm số `answer_relevancy` thấp một cách bất thường trên tất cả các nhóm phân phối (chỉ từ 0.16 đến 0.37). Nguyên nhân chủ đạo là do module đánh giá `m4_eval.py` đang sử dụng mô hình embedding tiếng Anh `all-MiniLM-L6-v2` để tính toán độ tương đồng cho các câu hỏi và câu trả lời bằng tiếng Việt, khiến kết quả similarity rất thấp và không chính xác. Ngoài ra, RAG pipeline có xu hướng trả về câu trả lời dài dòng, chứa nhiều đoạn dẫn nhập thừa thãi, làm giảm mật độ thông tin liên quan trực tiếp đến câu hỏi.

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating | Siết chặt system prompt, giảm temperature xuống 0.0, cải thiện chất lượng context. |
| context_recall | Missing relevant chunks | Bổ sung cơ chế tìm kiếm BM25 song song với Dense, tinh chỉnh lại kích thước chunk và chunk overlap. |
| context_precision | Too many irrelevant chunks | Sử dụng mô hình Re-ranking tốt hơn (như Cohere Rerank), lọc metadata theo phiên bản tài liệu. |
| answer_relevancy | Answer doesn't match question | Tinh chỉnh prompt sinh câu trả lời để yêu cầu trả lời trực tiếp và ngắn gọn; thay thế model embedding đánh giá bằng một mô hình đa ngôn ngữ tốt hơn (ví dụ: `BAAI/bge-m3`). |

---

## 6. Nhận xét về Adversarial Distribution

Điểm trung bình của nhóm `adversarial` (0.6966) thấp hơn nhóm `factual` (0.7356) nhưng lại cao hơn nhóm `multi_hop` (0.6185). Điều này đúng với kỳ vọng vì các câu hỏi adversarial chứa các bẫy về phiên bản tài liệu hoặc phủ định, gây khó khăn cho việc suy luận của LLM hơn là các câu factual thông thường. Tuy nhiên, nó vẫn dễ hơn nhóm multi-hop đòi hỏi phải tổng hợp thông tin từ nhiều nguồn tài liệu khác nhau. 

Có 2 câu hỏi trong Bottom 10 thuộc về nhóm adversarial (Rank 2 và Rank 4) liên quan đến chính sách VPN và tần suất thay đổi mật khẩu. RAG pipeline bị nhầm lẫn giữa các phiên bản quy định cũ (v1.0) và mới (v2.0) do thiếu cơ chế lọc metadata theo thời gian hoặc phiên bản tài liệu hoạt động hiệu quả.
