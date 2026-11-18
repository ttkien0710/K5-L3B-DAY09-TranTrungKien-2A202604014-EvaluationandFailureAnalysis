# Judge audit

Chạy: `python -m lab9.judge --csv <file> --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | Đạt (κ=0.757 ≥ 0.6, P=0.857 ≥ 0.8, R=0.800 ≥ 0.7) | dùng |
| helpful | 100 | 0.375 | 1.000 | 0.300 | Không đạt (κ=0.375 < 0.6, R=0.300 < 0.7) | loại |

## ≥5 case audit thủ công
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & Hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | H-008 (Tone) | PASS | FAIL | Người đúng | Câu: *"Em đã hoàn tiền cho anh rồi nha, anh check giúp em."* Xưng "em" và "nha" quá suồng sã, thiếu chuẩn mực CSKH B2C/C2C; Judge coi là thân thiện. **Sửa rubric:** Bổ sung quy định chuẩn xưng hô (phải xưng "LogiX" và gọi khách là "anh/chị"). |
| 2 | H-029 (Tone) | PASS | FAIL | Người đúng | Câu: *"Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm."* Giọng điệu gắt gỏng, mang tính ra lệnh đuổi khách; Judge chỉ nhìn thấy tiền tố "Dạ, vui lòng" nên nhầm là lịch sự. **Sửa rubric:** Cấm các cụm từ xua đuổi/ngăn khách phản hồi ("không cần nhắn thêm", "đừng hỏi nữa"). |
| 3 | H-042 (Tone) | FAIL | PASS | Người đúng | Câu: *"Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ."* Câu ngắn gọn, lịch sự, cung cấp đúng hạn tiền về; Judge phạt vì "thiếu lời xin lỗi". **Sửa rubric:** Làm rõ trong rubric rằng case giải quyết thành công không bắt buộc phải xin lỗi, chỉ cần lịch sự và rõ ràng. |
| 4 | HL-015 (Helpful) | PASS | FAIL | Người đúng | Reply: *"Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 ạ."* Nhưng ledger thực tế chỉ ghi 40.000đ (sai lệch 15.000đ). Judge chỉ đọc văn bản thấy mạch lạc nên chấm PASS. **Hành động:** Tiêu chí tiền và dữ liệu nghiệp vụ không được để LLM judge phỏng đoán, phải bắt buộc dùng code assertion đối chiếu ledger. |
| 5 | HL-016 (Helpful) | PASS | FAIL | Người đúng | Reply xin lỗi rất tha thiết, lịch sự nhưng nói hoàn 55.000đ trong khi ledger chỉ hoàn 35.000đ. Judge bị thiên kiến độ trôi chảy (fluency bias). **Hành động:** Loại bỏ `judge_helpful` khỏi release gate; thay thế bằng Trajectory Grader `RefundAmountCorrect` và Grader FM-08 bằng code. |

## Kết luận
- **Tiêu chí bị loại và lý do:**
  + Tiêu chí `helpful` bị **loại hoàn toàn** khỏi cổng release gate vì chỉ số độ tin cậy quá thấp: Cohen's Kappa chỉ đạt **0.375** (dưới sàn 0.6) và Recall chỉ đạt **0.300** (dưới sàn 0.7).
  + Ma trận nhầm lẫn của `helpful`: `{"tp": 9, "fp": 0, "fn": 21, "tn": 70}`. Có tới **21 ca False Negative (FN)**: con người đánh giá FAIL do agent nói sai số tiền so với sổ cái (ledger), nhưng LLM Judge lại chấm PASS vì câu từ trôi chảy, xin lỗi khách rất lịch sự.
  + LLM Judge không có khả năng tự kiểm chứng sự thật khách quan (ground truth trong database/ledger) nếu chỉ đọc text reply. Do đó, các tiêu chí liên quan đến tiền bạc, điều khoản và an toàn bắt buộc phải dùng Code Grader.
- **Rủi ro còn lại:**
  + LLM Judge `tone_ok` tuy đạt sàn kiểm chuẩn nhưng mới chỉ được đo lường trên 100 mẫu replay đã công khai (`judge_tone_public.csv`).
  + Khi agent gặp các mẫu câu mới ngoài đời thực (từ lóng, châm biếm tinh tế, teencode, ngôn ngữ vùng miền), Judge có thể phát sinh thêm lỗi. Cần thu thập dữ liệu người dùng thật và xây dựng tập held-out mới định kỳ để đánh giá lại calibration.

