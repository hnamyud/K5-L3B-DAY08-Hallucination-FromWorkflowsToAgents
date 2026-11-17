# Decision record

- Shape chọn: workflow
- Commit hash: 10fe29976d24b360f999cae949d0c64e2f977a79
- Run id dẫn chứng: R-workflow-C-001-s11; toàn bộ trial trong src/reports/benchmark.json
- Pass rate / pass^5 / seed min-max / variance: public 70/70 = 1.0 / 1.0 / 1.0–1.0 / 0.0
- Cost per success (mô phỏng, VND): 27.857142857142858
- Crash safety: 20/20 theo src/reports/crash.json; bộ student/reliability tích hợp 272 passed, gồm test crash của Workflow
- Secret leak / approval bypass: public Workflow 0/70 / 0/70 trial theo src/reports/public_safety.json; kiểm result, scanner-visible files, Python stdout/stderr và ledger/approval. Self-check student_drop: 200 trial Workflow, 0 error, 0 leak_suspect; không phải secret/approval counts của grader hidden và không thay thế xác nhận must-pass trên hidden

## Vì sao shape này

Chọn Workflow theo bằng chứng public trên cùng 14 claim × 5 seed (11,22,33,44,55), cùng MockLLM, pricing và evaluator, mỗi trial dùng SQLite mới. Workflow, Single và Orchestrator cùng pass 70/70, pass^5 = 1 và variance = 0, nhưng Workflow có cost/success thấp nhất: 27.857143 VND mô phỏng so với Single 46.802857, Orchestrator 47.852857 và Swarm 49.566957. Workflow không gọi model, phù hợp quy trình claim có luật xác định và đáp ứng mục tiêu chọn kiến trúc ít tự chủ nhất đủ làm việc. Đây là lựa chọn dựa trên public, chưa xác nhận ngưỡng hidden ≥ 0.75.

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean / p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | --- |
| workflow | 1.000000 | 1.000000 | 1–1 | 0 | 27.857143 | 1 / 1 |
| single | 1.000000 | 1.000000 | 1–1 | 0 | 46.802857 | 9.642857 / 11 |
| orchestrator | 1.000000 | 1.000000 | 1–1 | 0 | 47.852857 | 2.785714 / 3 |
| swarm | 0.985714 | 0.928571 | 0.928571–1 | 0.000816327 | 49.566957 | 2.8 / 3 |

## Có đáng đổi lấy multi-agent không

Trên corpus này, multi-agent tốn thêm chi phí mà chưa tăng pass rate. Swarm fail C-002 seed 55 vì tin số ngày trễ trong ticket. Ablation trong research card cho thấy việc bỏ cơ chế ghi đè sửa được lỗi này; kết quả chưa đủ để quy lỗi cho topology Swarm. Nhóm sẽ xem xét kiến trúc khác khi workflow cố định không xử lý được yêu cầu mới, hoặc khi thử nghiệm trên cùng dữ liệu cho thấy độ đúng hay khả năng xử lý tăng đủ để bù chi phí. Kiến trúc thay thế vẫn phải qua approval, secret và crash gates. Số lượng agent không phải tiêu chí để đổi kiến trúc.

## Trace và bằng chứng

Workflow ghi `quote` với amount_vnd, reason, clause_ids; `decision` cho needs_info/escalated; `error` khi claim không tồn tại; và `crash` tại điểm fault injection. Mỗi nhánh kết thúc dùng `finish()` để tạo output và ghi trace đúng contract. Khi có InjectedCrash, hàm ghi trace xuống file rồi truyền exception cho bộ test. Đầu vào đi qua get_claim_view/lookup_order, và trace bỏ các field nhạy cảm: claim/order row thô, ticket, parent_secret, raw headers, phone, địa chỉ, tên hàng và internal note. Evaluator kiểm file trace mới, run_id, events, ledger và chuỗi cấm. Code không tạo debug file ngoài phạm vi scanner. Nếu thêm log hoặc sink khác, nhóm cần audit riêng trước khi kết luận không rò rỉ. Trace không được commit; có thể tạo lại bằng benchmark.

## Tự kiểm student_drop

Đã chạy 40 claim không nhãn × 5 seed cho workflow, single và orchestrator (600 trial tổng cộng), lưu `src/reports/hidden_selfcheck.json`. Mỗi shape có 200 trial, 0 error và 0 leak_suspect; không báo lỗi trace. Không có nhãn nên graded_trials = 0, pass rate/pass^5/variance/cost per success đều null. Chưa xác nhận ngưỡng hidden ≥ 0.75 hoặc approval bypass trên hidden.

Lệnh tái lập trong `src/` (student_drop đặt ở gốc repo):

```powershell
python -m lab8.benchmark --shapes workflow,single,orchestrator --cases ../student_drop/hidden_claims.jsonl --world ../student_drop/hidden_world_state.json --seeds 11,22,33,44,55 --out reports/hidden_selfcheck.json
```

## Giới hạn

- Commit hash trên là revision code khi sinh benchmark, không tự nhận là commit nộp cuối. Sau khi chốt hồ sơ, nhóm cần làm theo thứ tự commit/hash trong SUBMISSION.md.
- MockLLM và giá mô phỏng không đại diện model hoặc hóa đơn production; latency là `unavailable`.
- Steps là số event trace, không phải số vòng ReAct hoặc latency; cách instrumentation khác nhau giữa shape nên không kết luận Workflow nhanh gấp số lần chênh steps.
- Crash suite chính thức chủ yếu kiểm host và một trường hợp Swarm; độ an toàn Workflow còn được kiểm bằng student/reliability tests, không suy ra từ riêng 20/20.
- Đã có safety counts public riêng trong public_safety.json; chưa có hidden grade hoặc kiểm định thống kê. Audit không bao phủ sink bên ngoài scanner hay output native process, không chứng minh an toàn trên mọi đầu vào.
- Các số liệu lấy từ report do công cụ sinh; không sửa tay JSON.
