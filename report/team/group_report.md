# Báo cáo nhóm — Lab 8 LogiX-VN

## 1. Thông tin nhóm

- Tên nhóm: Hallucination
- Thành viên: Nguyễn Duy Mạnh, Lê Hải Lân, Đỗ Triệu Đức Phương, Vũ Xuân Bách; thông tin và bằng chứng đóng góp trong `TEAM.md`.
- Repo: https://github.com/hnamyud/K5-L3B-DAY08-Hallucination-FromWorkflowsToAgents
- Revision code dùng cho benchmark: `fa0074022915797d867d6ac74e7ec8ad6c4ab8dd`

## 2. Mục tiêu và cách tiếp cận

Nhóm triển khai quy trình xử lý claim bằng Workflow, Single Agent và Orchestrator, rồi so sánh với Swarm có sẵn. Nhóm chọn kiến trúc ít tự chủ nhất vẫn xử lý đúng; hiệu quả của multi-agent phải được kiểm tra bằng số liệu. Số tiền và thao tác ghi refund do code quyết định. Order và policy hiện hành là nguồn dữ kiện chính; nội dung ticket cần được đối chiếu trước khi sử dụng.

Chia việc: Lê Hải Lân phụ trách Workflow và tích hợp; Đỗ Triệu Đức Phương phụ trách Single; Vũ Xuân Bách phụ trách Orchestrator/handoff; Nguyễn Duy Mạnh phụ trách reliability/evaluation và nghiên cứu. Nhóm kiểm baseline trước, triển khai ba shape song song, bổ sung kiểm thử crash/retry, tích hợp rồi benchmark. Mạnh thực hiện ablation độc lập trên Swarm trong lúc chờ tích hợp.

## 3. Giải pháp

### Workflow (A)

Workflow đọc claim bằng `get_claim_view`. Nếu thiếu order_id, hàm trả `needs_info`. Tiếp theo, `lookup_order` kiểm tra đơn theo principal; nếu không thấy đơn, kết quả là `escalated`. `compute_compensation` tính quote: bằng 0 thì trả `denied`, dương thì chuyển cho host. Luồng này không gọi model. Mỗi nhánh kết thúc dùng `finish()` để lấy ledger/audit thực và ghi trace. Trace chỉ ghi quote hoặc quyết định cần thiết, không dump dữ liệu thô.

### Single Agent (B)

Single dùng `run_react(max_steps=8, worker=True)` và tính mỗi MockLLM call vào budget. Vì MockLLM chỉ trả text, code chọn action theo observation, lần lượt gọi lookup_order, lookup_policy, lookup_refund_history và compute_compensation. Vòng lặp có dạng ReAct nhưng phần điều phối vẫn do code thực hiện, nên kết quả chưa đánh giá được khả năng model tự lập kế hoạch. Worker chỉ được dùng registry đọc. Sau vòng lặp, code chuyển quote đầy đủ cho host. Khi chạm step cap hoặc nhận kết quả bất hợp lệ, hàm trả `failed`. Với exception thông thường, trace chỉ ghi loại lỗi để tránh lộ dữ liệu; với InjectedCrash, hàm ghi trace xuống file rồi truyền exception ra ngoài.

### Orchestrator (C)

Orchestrator gọi MockLLM để parse/draft, dựng `make_handoff` với facts rỗng và allowlist tool đọc (`lookup_order`, `lookup_policy`, `compute_compensation`). `run_readonly_investigators` thực hiện đọc theo principal; worker không có quyền ghi refund. Trace ghi handoff đã sanitize, order view đã lọc và quote, không ghi parent context. Orchestrator dùng quote investigator rồi gọi host, không có cơ chế ghi đè ngày trễ từ ticket như điểm yếu Swarm.

### Tiền, durability và bảo mật

Ba shape giữ nguyên chữ ký `run_claim` và truyền **quote đầy đủ** vào `durability.issue_refund`. Host kiểm approval đúng claim, hiệu lực, giới hạn số tiền và khoản đã hoàn theo đơn; nhóm không tự trừ khoản cũ trước khi gọi host. Ledger dùng khóa `refund:{claim_id}`; effect và audit committed nằm trong một transaction. Shape truyền crash_at và để InjectedCrash thoát ra, nhờ đó test xác nhận crash thực sự xảy ra.

Handoff dùng allowlist, không dùng LLM summary làm sanitizer. Thiết kế tránh ghi parent_secret, raw headers, phone, địa chỉ, tên hàng và internal note vào trace/stdout. Các trial public của ba shape nhóm viết đã qua kiểm trace của evaluator: file mới của lần chạy, run_id khớp, events là list và không chứa chuỗi cấm theo nhãn public. Audit public độc lập trong `src/reports/public_safety.json` kiểm thêm kết quả, file thuộc phạm vi scanner, Python stdout/stderr và ledger/approval thực. Mỗi shape có 70 trial, secret_leak_count = 0, forbidden_leak_count = 0, approval_bypass_count = 0 và invalid_trace_count = 0. Đây không phải audit mọi output sink/đầu vào, không bao gồm output native process hoặc sink ngoài scanner, và không thay thế hidden report.

## 4. Kết quả

### Thiết lập và benchmark chính thức

Nguồn: `src/reports/benchmark.json`. Mỗi shape chạy 14 claim public × 5 seed `11,22,33,44,55` = 70 trial; cùng MockLLM, as_of, pricing, evaluator; SQLite mới cho từng trial. Các lệnh sau chạy trong `src/`:

```powershell
python -m pytest -q
python -m pytest -q -m student -o addopts=
python -m lab8.crash_test --count 20 --out reports/crash.json
python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
```

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean / p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | --- |
| workflow | 1.000000 | 1.000000 | 1–1 | 0 | 27.857143 | 1 / 1 |
| single | 1.000000 | 1.000000 | 1–1 | 0 | 46.802857 | 9.642857 / 11 |
| orchestrator | 1.000000 | 1.000000 | 1–1 | 0 | 47.852857 | 2.785714 / 3 |
| swarm | 0.985714 | 0.928571 | 0.928571–1 | 0.000816327 | 49.566957 | 2.8 / 3 |

Workflow/Single/Orchestrator pass 70/70. Swarm pass 69/70, fail C-002 seed 55 (`status resolved != denied`) do tin ngày trễ trong ticket. Run id dẫn chứng: `R-workflow-C-001-s11`, `R-swarm-C-002-s55`; report chứa run_id từng trial.

### Reliability và lựa chọn kiến trúc

Sau bản sửa test Orchestrator, suite mặc định có **63 passed**; sau khi thêm test audit safety, suite mặc định đạt **64 passed**. Sau merge bonus, lần kiểm lại có **68 passed, 272 deselected** ở suite mặc định và **272 passed, 68 deselected** ở suite student/reliability. `src/reports/crash.json` ghi **20/20**. Test TV4 bổ sung 24 trường hợp cho bốn shape: ba mốc crash × approval cap/order top-up, kiểm persisted state qua connection mới rồi resume hai lần; xác nhận quote đầy đủ, effect đúng và ledger/audit/checkpoint không bị nhân đôi.

Chọn **Workflow**: độ đúng public ngang Single/Orchestrator nhưng cost/success thấp nhất và không cần model. Crash suite có sẵn chủ yếu kiểm host và một trường hợp Swarm; bằng chứng Workflow còn dựa trên student/reliability tests. Audit safety public độc lập cho cả bốn shape không ghi nhận secret leak hoặc approval bypass; chạy lại bằng `python scripts/audit_public_safety.py` trong `src/`. Đã chạy self-check trên student_drop không nhãn: 40 claim × 5 seed × 3 shape = 600 trial; mỗi shape có 200 trial, error_trials = 0, leak_suspect_trials = 0 và không có lỗi trace được báo. Nguồn: `src/reports/hidden_selfcheck.json`. Mỗi shape trả 95 denied, 90 resolved, 5 needs_info và 10 escalated; đây là phân bố output, chưa phải nhãn đúng. Pass rate, pass^5, variance và cost/success đều null vì graded_trials = 0. Chưa có hidden grade chính thức, chưa xác nhận approval bypass trên hidden hoặc ngưỡng pass rate ≥ 0.75. Chi tiết quyết định và điều kiện đổi kiến trúc: `decision_adr.md`.

### Nghiên cứu R2 và ablation

Áp dụng MAST của Cemri et al. (2025) để đặt giả thuyết lỗi Swarm thuộc FC3 / FM-3.3 (Incorrect verification): đã có dữ kiện order nhưng lại dùng ticket. Đã đọc trace `R-swarm-C-002-s55`: handoff → investigator có order O-202 với delay_days=2 → model_override thành 5 → quote 20.000 VND theo SHIP-LATE-03. Trial benchmark trả resolved thay vì denied. Trace cho thấy dữ kiện đúng đã được lấy về rồi bị ghi đè. Nhóm dựa vào diễn biến này và định nghĩa trong Appendix A để gán FM-3.3. Nhãn do nhóm diễn giải, chưa được tác giả paper xác nhận và chưa có nghiên cứu agreement. Biến thể `bonus.swarm_no_override` chỉ vô hiệu hóa điều kiện override, đổi tên shape để tách trace; test source-diff kiểm phạm vi thay đổi. Baseline không bị sửa.

Nguồn: `src/reports/bonus_ablation.json`. Mỗi nhánh 70 trial; baseline 69/70 so với biến thể 70/70; pass^5 0.928571 → 1, variance 0.000816327 → 0, cost/success 49.566957 → 48.715714 VND mô phỏng. Chỉ C-002 seed 55 chuyển fail → pass; 69 cặp còn lại giữ nguyên pass. Model calls giữ nguyên, nhưng trial này giảm một tool call tính lại quote và cost 62.22 → 52.20. Hypothesis được ủng hộ trên public; không kết luận topology Swarm tự thân gây lỗi. Phương pháp, nguồn và lệnh tái lập: `research_card.md`. Đây là ablation cho bài chính, không tự nhận đã hoàn thành một topic bonus.

## 5. Khó khăn và cách xử lý

- Trước tích hợp, test reliability có 18 failed, 6 passed vì ba shape còn stub và chưa phát sinh InjectedCrash. Nhóm giữ nguyên assertion; sau tích hợp, suite student/reliability đạt 272 passed.
- `python -m pytest -q` từng có 1 failed, 62 passed ở test missing order của Orchestrator: test dùng C-009 và nhận denied thay vì needs_info. Dữ liệu public cho thấy C-009 có order O-404, còn C-004 mới thiếu order_id. TV3 sửa claim và comment sang C-004 tại `78c64d6`, giữ nguyên business logic và nhãn. Suite mặc định sau đó đạt 63 passed.
- Lần đầu truy cập paper, công cụ gặp lỗi DNS nên nhóm chưa điền kết luận từ nguồn. Sau khi arXiv truy cập lại được, nhóm đọc §3–5 và Appendix A của v3 để bổ sung claim/boundary.

## 6. Tổng hợp đóng góp

| Thành viên | Phần chính | Bằng chứng |
| --- | --- | --- |
| Lê Hải Lân | Workflow + integration | `workflow.py`; commit/test theo TEAM.md |
| Đỗ Triệu Đức Phương | Single Agent | `single.py`, `test_single_agent.py`; commit/test theo TEAM.md |
| Vũ Xuân Bách | Orchestrator + handoff; sửa test missing order | `orchestrator.py`, `test_orchestrator_student.py`; `78c64d6` |
| Nguyễn Duy Mạnh | Reliability/evaluation, ablation, metrics và research card | `test_evaluation_reliability.py`, `test_bonus_swarm_no_override.py`, crash/benchmark reports |

Bảng đóng góp đầy đủ trong `TEAM.md`; mỗi người hoàn thiện báo cáo cá nhân theo MSSV và kiểm tra tính chính xác của bằng chứng trước khi nộp. AI được dùng để hỗ trợ code, kiểm thử và soạn nội dung; công cụ và cách kiểm chứng ghi trong báo cáo cá nhân.

## 7. Hạn chế và hướng cải thiện

- MockLLM, corpus synthetic nhỏ và giá mô phỏng không đại diện hệ thống production; latency là `unavailable`. Chưa có kiểm định thống kê hoặc hidden grade.
- Steps đếm event trace; instrumentation của các shape khác nhau. Single có mean steps > 8 không đồng nghĩa vượt cap ReAct, và chênh steps không chứng minh chênh latency.
- Benchmark bốn shape thay cả luồng/prompt/tool calls, không phải thí nghiệm chỉ thay topology. Ablation riêng kiểm một cơ chế, nhưng prompt draft và chi phí downstream vẫn có thể đổi theo kết quả can thiệp.
- Nếu task mới đòi hỏi xử lý linh hoạt hoặc công việc song song thực sự, cần đo lại trước khi giữ/đổi quyết định Workflow. Không thêm agent chỉ vì kiến trúc phức tạp hơn.

## 8. Tài liệu tham khảo

- Cemri et al. (2025), *Why Do Multi-Agent LLM Systems Fail?*, arXiv:2503.13657v3: https://arxiv.org/html/2503.13657v3 ; §3–5, Appendix A.
- Tài liệu lab: `docs/business_rules.md`, `docs/contracts.md`, `docs/metrics.md`, `RULES.md`, `RUBRIC.md`, `SUBMISSION.md`.
- Bằng chứng: `src/reports/benchmark.json`, `src/reports/crash.json`, `src/reports/bonus_ablation.json`, `src/reports/public_safety.json`; quyết định/nghiên cứu: `decision_adr.md`, `research_card.md`.
- Audit safety public: `src/scripts/audit_public_safety.py`; test phản chứng: `src/tests/test_public_safety_audit.py`.
