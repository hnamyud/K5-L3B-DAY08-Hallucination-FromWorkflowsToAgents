# Báo cáo nhóm — Lab 8 LogiX-VN

## 1. Thông tin nhóm

- Tên nhóm: Hallucination; lớp khai báo trong `TEAM.md`: L3B.
- Thành viên: Nguyễn Duy Mạnh, Lê Hải Lân, Đỗ Triệu Đức Phương, Vũ Xuân Bách; thông tin và bằng chứng đóng góp trong `TEAM.md`.
- Repo: https://github.com/hnamyud/K5-L3B-DAY08-Hallucination-FromWorkflowsToAgents
- Revision code dùng cho benchmark: `10fe29976d24b360f999cae949d0c64e2f977a79`, trùng revision trong `decision_adr.md`; chưa tự nhận là commit nộp cuối. Nhánh nộp theo TEAM.md: `main`.

## 2. Mục tiêu và cách tiếp cận

Nhóm triển khai cùng quy trình xử lý claim theo Workflow, Single Agent và Orchestrator; dùng Swarm có sẵn làm đối chứng. Mục tiêu là chọn kiến trúc ít tự chủ nhất vẫn xử lý đúng, thay vì mặc định multi-agent tốt hơn. Model không quyết số tiền và không ghi refund; dữ kiện order và policy hiện hành là nguồn sự thật, chữ trong ticket là dữ liệu không đáng tin.

Chia việc: Lê Hải Lân phụ trách Workflow và tích hợp; Đỗ Triệu Đức Phương phụ trách Single; Vũ Xuân Bách phụ trách Orchestrator/handoff; Nguyễn Duy Mạnh phụ trách reliability/evaluation và nghiên cứu. Nhóm kiểm baseline trước, triển khai ba shape song song, bổ sung kiểm thử crash/retry, tích hợp rồi benchmark. TV4 thực hiện ablation độc lập trên Swarm trong lúc chờ tích hợp.

## 3. Giải pháp

### Workflow (A)

Pipeline cố định: đọc `get_claim_view` → thiếu order_id thì `needs_info` → `lookup_order` theo principal, không thấy đơn thì `escalated` → `compute_compensation` → quote bằng 0 thì `denied`, quote dương thì gọi host. Không có model call. Các nhánh kết thúc dùng `finish()` để lấy ledger/audit thực và tạo trace. Trace ghi quote hoặc quyết định ngắn gọn, không dump dữ liệu thô.

### Single Agent (B)

Dùng `run_react(max_steps=8, worker=True)`. Mỗi lần quyết định đều tính MockLLM call vào budget, nhưng MockLLM chỉ trả text nên code hiện tại chọn action tường minh theo observation: lookup_order → lookup_policy → lookup_refund_history → compute_compensation. Đây là vòng lặp ReAct-shaped có điều phối bằng code, không phải bằng chứng về khả năng model tự lập kế hoạch. Registry worker chỉ đọc; sau vòng lặp, code chuyển quote đầy đủ cho host. Step cap/kết quả bất hợp lệ trả `failed`; exception thông thường chỉ ghi loại lỗi để tránh lộ dữ liệu. InjectedCrash được flush trace rồi truyền ra ngoài.

### Orchestrator (C)

Orchestrator gọi MockLLM để parse/draft, dựng `make_handoff` với facts rỗng và allowlist tool đọc (`lookup_order`, `lookup_policy`, `compute_compensation`). `run_readonly_investigators` thực hiện đọc theo principal; worker không có quyền ghi refund. Trace ghi handoff đã sanitize, order view đã lọc và quote, không ghi parent context. Orchestrator dùng quote investigator rồi gọi host, không có cơ chế ghi đè ngày trễ từ ticket như điểm yếu Swarm.

### Tiền, durability và bảo mật

Ba shape giữ nguyên chữ ký `run_claim` và truyền **quote đầy đủ** vào `durability.issue_refund`. Host kiểm approval đúng claim, hiệu lực, giới hạn số tiền và khoản đã hoàn theo đơn; nhóm không tự trừ khoản cũ trước khi gọi host. Ledger dùng khóa `refund:{claim_id}`; effect và audit committed nằm trong một transaction. Shape truyền crash_at và để InjectedCrash thoát ra, nhờ đó test xác nhận crash thực sự xảy ra.

Handoff dùng allowlist, không dùng LLM summary làm sanitizer. Không ghi parent_secret, raw headers, phone, địa chỉ, tên hàng, internal note vào trace/stdout. Trace được kiểm đúng contract: file mới của lần chạy, run_id khớp và events là list. Không sửa policy, durability, metrics, config/data hoặc sửa tay JSON báo cáo.

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

Sau bản sửa test TV3, suite mặc định có **63 passed**, suite student/reliability có **272 passed**. `src/reports/crash.json` ghi **20/20**. Test TV4 bổ sung 24 trường hợp cho bốn shape: ba mốc crash × approval cap/order top-up, kiểm persisted state qua connection mới rồi resume hai lần; xác nhận quote đầy đủ, effect đúng và ledger/audit/checkpoint không bị nhân đôi.

Chọn **Workflow**: độ đúng public ngang Single/Orchestrator nhưng cost/success thấp nhất và không cần model. Crash suite có sẵn chủ yếu kiểm host và một trường hợp Swarm; bằng chứng Workflow còn dựa trên student/reliability tests. Chưa có hidden report, self-check hidden hoặc safety counts độc lập; không suy ra đã đạt ngưỡng hidden hay mọi must-pass chỉ từ pass rate public. Chi tiết quyết định và điều kiện đổi kiến trúc: `decision_adr.md`.

### Nghiên cứu R2 và ablation

Áp dụng MAST của Cemri et al. (2025) để đặt giả thuyết lỗi Swarm thuộc FC3 / FM-3.3 (Incorrect verification): đã có dữ kiện order nhưng lại dùng ticket. Đây là ánh xạ đề xuất của nhóm, không phải nhãn tác giả gán cho lab. Biến thể `bonus.swarm_no_override` chỉ vô hiệu hóa điều kiện override, đổi tên shape để tách trace; test source-diff kiểm phạm vi thay đổi. Baseline không bị sửa.

Nguồn: `src/reports/bonus_ablation.json`. Mỗi nhánh 70 trial; baseline 69/70 so với biến thể 70/70; pass^5 0.928571 → 1, variance 0.000816327 → 0, cost/success 49.566957 → 48.715714 VND mô phỏng. Chỉ C-002 seed 55 chuyển fail → pass; 69 cặp còn lại giữ nguyên pass. Model calls giữ nguyên, nhưng trial này giảm một tool call tính lại quote và cost 62.22 → 52.20. Hypothesis được ủng hộ trên public; không kết luận topology Swarm tự thân gây lỗi. Phương pháp, nguồn và lệnh tái lập: `research_card.md`. Đây là ablation cho bài chính, không tự nhận đã hoàn thành một topic bonus.

## 5. Khó khăn và cách xử lý

- **Shape stub trước tích hợp:** test reliability từng có 18 failed, 6 passed vì ba shape không phát sinh InjectedCrash. Không bỏ/skip assertion; sau tích hợp, cả suite student/reliability đạt 272 passed.
- **Sai dữ liệu test Orchestrator:** `python -m pytest -q` từng có 1 failed, 62 passed: test missing order dùng C-009, nhận denied thay vì needs_info. Đối chiếu public thấy C-009 có order O-404; C-004 mới thiếu order_id. TV3 sửa test/comment sang C-004 tại `78c64d6`, không sửa business logic hoặc nhãn để chiều test. Suite mặc định sau đó đạt 63 passed.
- **Không truy cập được nguồn paper lúc đầu:** công cụ báo lỗi DNS; nhóm để trống kết luận chưa xác minh, không bịa nội dung. Khi arXiv truy cập lại được, đã đọc §3–5 và Appendix A của v3 để cập nhật claim/boundary.

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
- Trước nộp cần chốt commit/hash, hoàn thiện hồ sơ, tự kiểm hidden khi được cấp và chạy `check_submission` đạt 0 FAIL. Các thay đổi code sau revision đã đo phải được kiểm và sinh report lại.
- Tên lớp/repo hiện khai báo L3B, trong khi SUBMISSION.md của starter quy định L3A; nhóm cần xác nhận với coach hoặc sửa theo quy định trước khi nộp. Báo cáo giữ nguyên thông tin nhóm đang khai báo, không tự đổi lớp.

## 8. Tài liệu tham khảo

- Cemri et al. (2025), *Why Do Multi-Agent LLM Systems Fail?*, arXiv:2503.13657v3: https://arxiv.org/html/2503.13657v3 ; §3–5, Appendix A.
- Tài liệu lab: `docs/business_rules.md`, `docs/contracts.md`, `docs/metrics.md`, `RULES.md`, `RUBRIC.md`, `SUBMISSION.md`.
- Bằng chứng: `src/reports/benchmark.json`, `src/reports/crash.json`, `src/reports/bonus_ablation.json`; quyết định/nghiên cứu: `decision_adr.md`, `research_card.md`.
