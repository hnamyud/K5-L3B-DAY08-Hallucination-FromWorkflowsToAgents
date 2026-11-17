# Decision record

- Shape chọn: workflow
- Commit hash: 10fe29976d24b360f999cae949d0c64e2f977a79
- Run id dẫn chứng: R-workflow-C-001-s11; toàn bộ trial trong src/reports/benchmark.json
- Pass rate / pass^5 / seed min-max / variance: public 70/70 = 1.0 / 1.0 / 1.0–1.0 / 0.0
- Cost per success (mô phỏng, VND): 27.857142857142858
- Crash safety: 20/20 theo src/reports/crash.json; bộ student/reliability tích hợp 272 passed, gồm test crash của Workflow
- Secret leak / approval bypass: chưa có safety counts độc lập hoặc hidden report; 70/70 trial Workflow qua evaluator public, không thay thế xác nhận must-pass trên hidden

## Vì sao shape này

Chọn Workflow theo bằng chứng public trên cùng 14 claim × 5 seed (11,22,33,44,55), cùng MockLLM, pricing và evaluator, mỗi trial dùng SQLite mới. Workflow, Single và Orchestrator cùng pass 70/70, pass^5 = 1 và variance = 0, nhưng Workflow có cost/success thấp nhất: 27.857143 VND mô phỏng so với Single 46.802857, Orchestrator 47.852857 và Swarm 49.566957. Workflow không gọi model, phù hợp quy trình claim có luật xác định và đáp ứng mục tiêu chọn kiến trúc ít tự chủ nhất đủ làm việc. Đây là lựa chọn dựa trên public, chưa xác nhận ngưỡng hidden ≥ 0.75.

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean / p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | --- |
| workflow | 1.000000 | 1.000000 | 1–1 | 0 | 27.857143 | 1 / 1 |
| single | 1.000000 | 1.000000 | 1–1 | 0 | 46.802857 | 9.642857 / 11 |
| orchestrator | 1.000000 | 1.000000 | 1–1 | 0 | 47.852857 | 2.785714 / 3 |
| swarm | 0.985714 | 0.928571 | 0.928571–1 | 0.000816327 | 49.566957 | 2.8 / 3 |

## Có đáng đổi lấy multi-agent không

Trên corpus này, chưa có mức tăng pass rate để bù chi phí của multi-agent. Swarm còn fail C-002 seed 55 do tin số ngày trễ trong ticket. Ablation riêng trong research card hỗ trợ nhận định đây là lỗi cơ chế kiểm chứng dữ kiện, không chứng minh topology Swarm tự thân gây lỗi. Sẽ xem xét lại quyết định nếu corpus mới có yêu cầu mà workflow cố định không xử lý được, hoặc thử nghiệm kiểm soát trên cùng dữ liệu cho thấy kiến trúc khác cải thiện đủ lớn về độ đúng/khả năng xử lý mà vẫn qua approval, secret và crash gates với chi phí chấp nhận được. Không đổi kiến trúc chỉ vì nhiều agent hơn.

## Trace và bằng chứng

Workflow ghi `quote` (amount_vnd, reason, clause_ids), `decision` cho needs_info/escalated, `error` khi claim không tồn tại và `crash` tại điểm fault injection. `finish()` tạo output và ghi trace đúng contract trên các nhánh kết thúc; khi InjectedCrash xảy ra, trace được flush rồi exception truyền ra cho bộ test. Không ghi claim/order row thô, ticket, parent_secret, raw headers, phone, địa chỉ, tên hàng hoặc internal note. Các field đầu vào đi qua get_claim_view/lookup_order. Trace có run_id và events; evaluator kiểm file mới, run_id, ledger và chuỗi cấm. Không tạo debug file ngoài phạm vi scanner; nếu thêm log/sink khác, phải audit riêng trước khi tuyên bố không rò rỉ. Trace không commit; tái tạo bằng chạy benchmark.

## Giới hạn

- Commit hash trên là revision code khi sinh benchmark, không tự nhận là commit nộp cuối. Sau khi chốt hồ sơ, nhóm cần làm theo thứ tự commit/hash trong SUBMISSION.md.
- MockLLM và giá mô phỏng không đại diện model hoặc hóa đơn production; latency là `unavailable`.
- Steps là số event trace, không phải số vòng ReAct hoặc latency; cách instrumentation khác nhau giữa shape nên không kết luận Workflow nhanh gấp số lần chênh steps.
- Crash suite chính thức chủ yếu kiểm host và một trường hợp Swarm; độ an toàn Workflow còn được kiểm bằng student/reliability tests, không suy ra từ riêng 20/20.
- Chưa có hidden grade, safety counts riêng hay kiểm định thống kê; không ghi approval_bypass_count/secret_leak_count bằng 0 khi chưa có report tương ứng.
- Các số liệu lấy từ report do công cụ sinh; không sửa tay JSON.
