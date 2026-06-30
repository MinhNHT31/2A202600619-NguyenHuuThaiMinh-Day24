# LLM Judge Bias Report — Phase B

**Sinh viên:** Nguyễn Hữu Thái Minh  
**Ngày:** 30/06/2026  
**Judge model:** openai/gpt-4o-mini

---

## 1. Pairwise Judge Results

*(Chạy pairwise_judge() trên 10 cặp answers)*

| # | Question (tóm tắt) | Winner | Reasoning tóm tắt |
|---|---|---|---|
| 1 | Nghỉ phép khi kết hôn | A | A đầy đủ hơn khi nêu rõ nghỉ kết hôn không bị trừ vào phép năm. |
| 5 | Phê duyệt thiết bị 55tr | A | A chính xác vì chỉ ra đơn trên 50 triệu cần CEO duyệt (B sai khi bảo Director duyệt). |
| 12 | Thưởng Tết tối thiểu | A | A đầy đủ hơn vì bổ sung thông tin thưởng pro-rata cho nhân viên làm việc dưới 6 tháng. |
| 21 | Nghỉ phép & Lương Senior 9 năm | tie | Cả hai câu trả lời đều đúng về số ngày phép (18) và khoảng lương (20-35 triệu). |
| 23 | Hoàn trả học phí sau 8 tháng | A | A giải thích chi tiết hơn về thời gian cam kết tối thiểu 1 năm và lý do hoàn trả 100%. |
| 29 | Tạm ứng 8tr trễ hạn 30 ngày | A | A tính toán chính xác phí phạt quá hạn 15 ngày (80k) và quy trình duyệt phức tạp hơn B. |
| 33 | Manager 12 năm: phép & phụ cấp | A | A giải thích chi tiết cơ cấu tính phụ cấp (ăn trưa + điện thoại) và cách tính ngày phép thâm niên. |
| 41 | Ngày phép năm cơ bản | A | A chỉ rõ sự khác biệt giữa chính sách v2024 (15 ngày) và v2023 cũ (12 ngày). |
| 46 | Thử việc được nghỉ phép không | A | A giải thích chi tiết hơn về quy trình nghỉ không lương đối với nhân viên thử việc. |
| 50 | VPN cá nhân khi WFH | A | A chính xác khi chỉ ra VPN cá nhân bị cấm và bắt buộc dùng VPN WireGuard (B sai khi cho phép). |

---

## 2. Swap-and-Average Results

*(Chạy swap_and_average() trên cùng các cặp)*

| # | Pass 1 Winner | Pass 2 Winner | Final | Position Consistent? |
|---|---|---|---|---|
| 1 | A | A | A | Yes |
| 5 | A | A | A | Yes |
| 12 | A | A | A | Yes |
| 21 | tie | A | tie | No |
| 23 | A | A | A | Yes |
| 29 | A | A | A | Yes |
| 33 | A | A | A | Yes |
| 41 | A | A | A | Yes |
| 46 | A | A | A | Yes |
| 50 | A | A | A | Yes |

**Position bias rate:** 10.0% (1/10 cases NOT consistent)

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu, 5 label=1, 5 label=0)  
**Judge labels:** `[0, 0, 0, 1, 0, 0, 0, 0, 0, 0]`

| Question ID | Human Label | Judge Label | Agree? |
|---|---|---|---|
| 1 | 1 | 0 | No |
| 5 | 0 | 0 | Yes |
| 12 | 1 | 0 | No |
| 21 | 1 | 1 | Yes |
| 23 | 1 | 0 | No |
| 29 | 0 | 0 | Yes |
| 33 | 1 | 0 | No |
| 41 | 0 | 0 | Yes |
| 46 | 1 | 0 | No |
| 50 | 0 | 0 | Yes |

**Cohen's κ:** 0.138  
**Interpretation:** slight agreement

---

## 4. Verbosity Bias

Trong các case có winner rõ ràng (không phải tie):
- A thắng + A dài hơn B: 9 / 9 cases
- B thắng + B dài hơn A: 0 / 9 cases  
- **Verbosity bias rate:** 100.0%

**Kết luận:** LLM Judge (openai/gpt-4o-mini) thể hiện xu hướng cực kỳ mạnh mẽ (100% trong các case thắng tuyệt đối) trong việc lựa chọn câu trả lời dài hơn và chi tiết hơn (Answer A - Ground Truth). Đây là một vấn đề nghiêm trọng (Verbosity Bias) vì trong nhiều trường hợp thực tế, một câu trả lời súc tích, ngắn gọn (Answer B) vẫn chính xác và đầy đủ, nhưng LLM vẫn ưu tiên Answer A chỉ vì nó chứa nhiều thông tin phụ trợ hoặc giải thích dài dòng hơn. Điều này làm giảm tính khách quan của hệ thống đánh giá tự động.

---

## 5. Nhận xét chung

* Hệ số κ đạt 0.138 (mức độ đồng thuận slight agreement - rất thấp). LLM Judge tại thời điểm hiện tại chưa đủ tin cậy để thay thế hoàn toàn con người vì nó quá khắt khe đối với các câu trả lời ngắn gọn và bị ảnh hưởng nặng nề bởi độ dài câu chữ (verbosity).
* Tỷ lệ position bias chỉ là 10.0% (1/10 cases). Đây là mức rất thấp và không quá đáng lo ngại đối với `openai/gpt-4o-mini`, cho thấy mô hình giữ được sự ổn định tương đối về mặt lập luận bất kể thứ tự trình bày.
* Cơ chế `swap-and-average` thực sự hữu ích. Nó giúp phát hiện ra trường hợp không nhất quán ở Q#21 (lần 1 chọn tie, lần 2 chọn A) và tự động đưa ra kết quả an toàn là "tie" để tránh thiên vị vị trí.
* Trong môi trường production, để sử dụng LLM Judge hiệu quả, chúng ta cần:
  1. Cung cấp hướng dẫn prompt cực kỳ chi tiết, giới hạn chặt chẽ tiêu chí về độ dài (ví dụ: trừ điểm nếu trả lời rườm rà).
  2. Sử dụng cơ chế `swap-and-average` bắt buộc.
  3. Định kỳ đối chiếu với một tập dữ liệu nhỏ có nhãn con người để kiểm tra độ tin cậy.
