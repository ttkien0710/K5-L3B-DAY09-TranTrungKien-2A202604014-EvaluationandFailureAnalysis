# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

Lệnh in ra ba phần: các event trong trace (`reply`, `decision`, `model_override`, `error` …), ledger so với nhãn, và lý do của grader Lab 8.

## Mẫu điền (mỗi bạn 1 case, lấy từ trial đang fail trong report)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11`, run ID: `seeded_regression-C-002-11` |
| Triệu chứng | `TrialPass` (lab8 grader) fail (`status resolved != denied`); `DbFactKept` fail; `RefundAmountCorrect` fail (`ledger 20000 != expected 0`). Reply nói: *"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."* trong khi ledger ghi nhận hoàn 20.000đ (nhãn kỳ vọng không hoàn tiền: `refund_vnd=0`, `status=denied`). |
| Bước sai đầu tiên | Event 3: `model_override` với payload `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Event 1 (`handoff`) và Event 2 (`investigator`) hoàn toàn đúng: DB trả về sự thật khách quan `delay_days=2`. Bước 3 agent tin ticket tự ý override thành 5 ngày, làm kéo theo Event 4 (`quote`) và Event 6 (`reply`) sai dây chuyền. |
| Failure mode | `FM-03` (ticket_trust — tin ticket hơn DB) |
| Đối chứng | Cùng `C-002@seed 11` trên shape `fixed`: Không có event `model_override`. Agent giữ nguyên `delay_days=2` của DB, quote tính ra `amount_vnd=0`, từ chối hoàn tiền chuẩn xác (`PASS`). |
| Regression test | `test_override_event_breaks_db_fact` (`DbFactKept`) và `test_wrong_amount_in_ledger` (`RefundAmountCorrect`) trong `tests/test_lab9_graders.py`. Đã xác minh fail trên `seeded_regression` và pass trên `fixed`. |
| Chưa chắc chắn | Trace chỉ ghi lại hành vi override của model, chưa phản ánh được cơ chế suy luận bên trong prompt khiến LLM ưu tiên thông tin khiếu nại hơn DB. |


## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
