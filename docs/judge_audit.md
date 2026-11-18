# Judge audit

Chạy: `python -m lab9.judge --csv data/eval/judge_tone_public.csv --audit 5` và `python -m lab9.judge --csv data/eval/judge_helpful_public.csv --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | Có (κ=0.757 ≥ 0.6, P=0.857 ≥ 0.8, R=0.800 ≥ 0.7) | **dùng** |
| helpful | 100 | 0.375 | 1.000 | 0.300 | Không (κ=0.375 < 0.6, R=0.300 < 0.7) | **loại** |

## ≥5 case audit thủ công

| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | `H-008` (tone) | PASS | FAIL | Người đúng | `"Em đã hoàn tiền cho anh rồi nha, anh check giúp em."` — Xưng "em" + "nha" quá suồng sã với khách; judge chỉ chấm thân thiện. **Sửa rubric:** quy định rõ xưng hô chuẩn `LogiX` / `anh/chị`, cấm từ ngữ suồng sã (`nha`, `nhé`, `check`). |
| 2 | `H-029` (tone) | PASS | FAIL | Người đúng | `"Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm."` — Có `"Dạ, vui lòng"` nên judge cho PASS, nhưng câu cuối mang giọng ra lệnh. **Sửa rubric:** bổ sung ví dụ phản hồi bị động/ra lệnh (`không cần nhắn thêm`) là FAIL bất kể có từ lịch sự ở đầu. |
| 3 | `H-042` (tone) | FAIL | PASS | Người đúng | `"Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ."` — Ngắn gọn nhưng đầy đủ, lịch sự; judge tự suy diễn phạt vì "thiếu xin lỗi". **Sửa rubric:** nêu rõ chỉ yêu cầu xin lỗi khi từ chối hoặc trễ/lỗi hệ thống, không phạt câu thông báo hoàn tiền thành công. |
| 4 | `H-051` (tone) | FAIL | PASS | Người đúng | `"Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ."` — CS LogiX cho phép dùng từ `"refund"`, judge phạt vì lẫn tiếng Anh. **Sửa rubric:** thêm danh sách thuật ngữ tiếng Anh được chấp nhận (`refund`, `claim`, `order`). |
| 5 | `H-081` (tone) | FAIL | PASS | Judge đúng | `"Dạ, đơn O-107 chưa đủ điều kiện hoàn theo chính sách hiện hành ạ."` — Câu trả lời lịch sự chuẩn; nhãn người bị gán nhầm do đọc nhầm ticket. **Hành động:** sửa lại gold label của `H-081` thành `PASS`. |
| 6 | `HL-015` (helpful) | PASS | FAIL | Người đúng | `"Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 ạ."` (ledger ghi `40.000đ`). Judge chỉ đọc reply trôi chảy mà không thấy ledger nên bỏ sót toàn bộ lỗi sai tiền (FM-08). **Hành động:** không dùng judge cho tính đúng đắn số tiền; thay bằng code grader `ReplyAmountMatchesLedger` đối chiếu trực tiếp `snapshot["refund_vnd"]`. |
| 7 | `HL-016` (helpful) | PASS | FAIL | Người đúng | Reply xin lỗi rất dài dòng, lịch sự nhưng báo sai số tiền (`55.000đ` vs ledger `35.000đ`). Judge bị đánh lừa bởi độ dài và giọng xin lỗi (verbosity/politeness bias). |

## Kết luận
- **Tiêu chí bị loại:** `helpful` — bị loại vì `κ = 0.375 < 0.6` và `Recall = 0.300 < 0.7`. Từ ma trận nhầm lẫn (`TP=9, FP=0, FN=21, TN=70`), judge bỏ sót 21/30 ca FAIL (`FN=21`), chủ yếu do các reply báo sai số tiền so với ledger (`HL-015`, `HL-016`, `HL-020`, `HL-028`, ...) nhưng viết trôi chảy, lịch sự nên judge đều chấm `PASS`. Không được dùng judge này làm gate; việc kiểm tra số tiền phải dùng code grader (`RefundAmountCorrect`, `ReplyAmountMatchesLedger`).
- **Tiêu chí được dùng:** `tone_ok` (`TP=24, FP=4, FN=6, TN=66`, `κ=0.757, P=0.857, R=0.800`) đạt cả 3 ngưỡng FLOORS, có thể dùng để theo dõi văn phong sau khi bổ sung các quy tắc xưng hô/ngữ cảnh vào rubric.
- **Rủi ro còn lại:** judge chỉ được kiểm trên dữ liệu replay đã công khai; cần nhãn người mới để có tập held-out thật trước khi dùng cho dữ liệu production.
