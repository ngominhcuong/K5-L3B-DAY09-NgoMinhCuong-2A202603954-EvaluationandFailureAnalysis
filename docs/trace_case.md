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
| Trial | `seeded_regression C-002@11` (`E-002@11`), run ID `E-seeded_regression-20261118-9dd7` |
| Triệu chứng | `db_fact_kept=False`, `refund_amount_correct=False`, `pass_rate=False`; reply nói `"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."` và ledger ghi hoàn `20.000đ` trong khi nhãn kỳ vọng `status="denied"`, `refund_vnd=0`. |
| Bước sai đầu tiên | Event `#2` — `kind: model_override`, payload `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Event `#0` (`handoff`) giao đúng nhiệm vụ `check_applicable_policy` và danh sách tool cho phép; Event `#1` (`investigator`) truy vấn `lookup_order` trả về đúng `O-202` với `delay_days=2` từ DB. Chỉ tới Event `#2`, agent mới lấy `parsed_delay=5` từ ticket để ghi đè lên dữ liệu DB. |
| Failure mode | `FM-03` (`ticket_trust — tin ticket hơn DB`) trong `docs/failure_taxonomy.md` |
| Đối chứng | Cùng `C-002@11` trên shape `fixed` (run ID `E-fixed-20261118-9dd7`): sau `investigator`, agent giữ nguyên `db_delay=2`, không phát sinh event `model_override`, `quote` trả `0đ` và `reply` từ chối đúng chính sách (`pass_rate=True`). |
| Regression test | `test_override_event_breaks_db_fact` & `test_wrong_amount_in_ledger` trong `src/tests/test_lab9_graders.py` (`DbFactKept` + `RefundAmountCorrect`), cùng item `E-002` (`FM-03`) trong `src/data/eval/eval_set_v1.jsonl`. Đã thấy fail trên `seeded_regression` (`db_fact_kept=False`, `refund_amount_correct=False`) và pass trên `fixed` (`True`). |
| Chưa chắc chắn | Trace trên MockLLM không phản ánh xác suất model thật có tự sửa lại sau khi đọc policy hay không; ngoài ra biến thể không ghi event `model_override` (`quiet_override`) sẽ không bị `DbFactKept` bắt ở bước này mà chỉ bị `RefundAmountCorrect` bắt ở ledger. |

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
