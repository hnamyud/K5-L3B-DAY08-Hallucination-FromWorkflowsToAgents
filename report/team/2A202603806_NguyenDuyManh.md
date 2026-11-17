# Báo cáo cá nhân — 2A202603806_NguyenDuyManh

Báo cáo phần Reliability & Evaluation đã thực hiện trên nhánh `feature/evaluation`. Đã kiểm tra ba shape sau tích hợp và đối chiếu benchmark chính thức bốn shape tại revision `10fe299`. ADR chọn Workflow theo số liệu public; hidden và hồ sơ nộp cuối vẫn cần kiểm tra.

## 1. Thông tin thành viên

- Họ tên: Nguyễn Duy Mạnh
- MSSV: 2A202603806
- Email: 26ai.manhnd9@vinuni.edu.vn
- GitHub: hnamyud

## 2. Phần việc phụ trách

Reliability & Evaluation: bổ sung test crash/retry, kiểm tra approval cap và trần hoàn theo đơn; chạy crash suite; nghiên cứu taxonomy MAST, thực hiện ablation Swarm và tổng hợp research card. Phụ trách benchmark bốn shape, phân tích metrics và ADR sau khi tích hợp đủ các shape.

## 3. Đóng góp cụ thể

- Thêm `src/tests/test_evaluation_reliability.py`: 24 trường hợp cho bốn shape, kết hợp ba mốc crash (`before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`) với approval cap (quote 35.000 VND, effect 20.000 VND) và order top-up (quote 150.000 VND, effect 120.000 VND).
- Kiểm tra trạng thái đã lưu qua connection mới sau crash; resume hai lần để xác nhận không nhân đôi ledger, audit và checkpoint. Dùng evaluator có sẵn để đối chiếu output, ledger và trace.
- Thêm `src/bonus/swarm_no_override.py`: bản sao thí nghiệm chỉ tắt điều kiện ghi đè ngày trễ từ ticket, đổi tên shape để tách trace; giữ nguyên Swarm tham chiếu và host.
- Thêm `src/tests/test_bonus_swarm_no_override.py`: hai trường hợp kiểm dữ kiện order và một test source-diff xác nhận phạm vi thay đổi của hàm `_run` so với baseline.
- Chạy ablation trên 14 claim public × 5 seed cho mỗi biến thể, dùng benchmark/evaluator chính thức và SQLite mới cho từng trial. Không sửa tay JSON kết quả.
- Đọc các phần liên quan của paper MAST (§3–5 và Appendix A), điền nguồn, kết luận, giới hạn và ánh xạ lỗi dự kiến vào `report/team/research_card.md`. Phân biệt giả thuyết với kết quả đo và không suy rộng sang model sản xuất.
- Bổ sung `src/scripts/audit_public_safety.py` và `src/tests/test_public_safety_audit.py`: audit 280 trial public, kiểm result, file thuộc scanner, Python stdout/stderr và ledger/approval thực. Test phản chứng xác nhận phát hiện secret chèn vào stdout và refund có approval bị thu hồi.
- Commit kỹ thuật: `8778b03` — `test: add reliability checks and controlled Swarm ablation`. Hash này dẫn chứng phần reliability/ablation ban đầu, không đại diện các thay đổi audit safety đang chờ chốt commit.

## 4. Bằng chứng

### Kiểm thử

Các lệnh tái lập dưới đây chạy trong `src/`. Các lệnh công cụ được chạy với interpreter `.venv` của repo và cấu hình pytest tương ứng.

| Lệnh | Kết quả đã ghi nhận |
| --- | --- |
| `python -m lab8.smoke` | `lab8.smoke pass` (thành viên tự chạy) |
| `python -m pytest -q` | 57 passed, 272 deselected; gồm 54 test host có sẵn và 3 test ablation mới |
| `python -m pytest -q -m student -o addopts= tests/test_evaluation_reliability.py -k swarm` | 6 passed, 18 deselected |
| `python -m pytest -q -m student -o addopts= -k swarm` | 68 passed, 261 deselected sau khi thêm test ablation |
| `python -m pytest -q -m student -o addopts= tests/test_evaluation_reliability.py` | 18 failed, 6 passed tại thời điểm ba shape còn stub; cả 18 lỗi là không phát sinh `InjectedCrash` |
| `python -m lab8.crash_test --count 20 --out reports/crash.json` | `crash_safety 20/20` (thành viên tự chạy; đã đối chiếu report) |

Bằng chứng crash: `src/reports/crash.json`, 20/20 scenario có `passed=true`. Suite chính thức kiểm host và một trường hợp Swarm, không chứng minh ba shape stub đã an toàn. Test reliability phải chạy lại trên từng shape sau khi nhận code.

### Ablation

Lệnh tái lập:

```powershell
python -m lab8.benchmark --shapes swarm,swarm_no_override --extra-shape swarm_no_override=bonus.swarm_no_override:run_claim --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/bonus_ablation.json
```

File report: `src/reports/bonus_ablation.json`. Cặp run id dẫn chứng: `R-swarm-C-002-s55` và `R-swarm_no_override-C-002-s55`.

| Metric | Swarm gốc | Swarm tắt override |
| --- | ---: | ---: |
| Trial pass | 69/70 | 70/70 |
| Pass rate | 0.985714 | 1.000000 |
| Pass^5 | 0.928571 | 1.000000 |
| Seed variance | 0.000816327 | 0 |
| Cost/success (VND mô phỏng) | 49.566957 | 48.715714 |

Chỉ C-002 seed 55 chuyển fail → pass (resolved → denied); 69 cặp còn lại giữ nguyên pass. Kết quả ủng hộ hypothesis trên public: không tin số ngày trễ trong ticket giúp loại bỏ điểm yếu đã công bố của baseline. Không có bằng chứng để kết luận topology Swarm tự thân gây lỗi hoặc mọi hệ multi-agent kém hơn workflow.

Model calls giữ nguyên nhưng tool calls ở trial này giảm 6 → 5 do không còn tính lại quote; cost giảm 62.22 → 52.20 VND mô phỏng. Đây là hệ quả của can thiệp, không phải mọi lời gọi và nội dung prompt đều bất biến. Latency là `unavailable`; chưa có kiểm định thống kê, hidden hoặc safety counts riêng cho biến thể.

### Kiểm tra sau tích hợp

Sau khi merge bản sửa test TV3 tại revision `10fe299`, kiểm thử tích hợp cho kết quả **63 passed, 272 deselected** ở suite mặc định và **272 passed, 63 deselected** ở suite student/reliability. Các kết quả trước tích hợp ở bảng trên là lịch sử, không phải trạng thái hiện tại.

Thành viên đã chạy lại crash suite đạt **20/20** và benchmark chính thức bốn shape; đã đối chiếu `src/reports/benchmark.json` (70 trial mỗi shape):

| Shape | Pass rate | Pass^5 | Variance | Cost/success (VND mô phỏng) |
| --- | ---: | ---: | ---: | ---: |
| workflow | 1 | 1 | 0 | 27.857143 |
| single | 1 | 1 | 0 | 46.802857 |
| orchestrator | 1 | 1 | 0 | 47.852857 |
| swarm | 0.985714 | 0.928571 | 0.000816327 | 49.566957 |

Đã soạn `decision_adr.md` chọn Workflow: cùng pass rate với Single/Orchestrator, chi phí mô phỏng thấp nhất và ít tự chủ nhất. Run id dẫn chứng: `R-workflow-C-001-s11`. Chưa xác nhận hidden ≥ 0.75. Đã có safety counts public độc lập như phần audit dưới đây, không phải safety counts hidden. Steps là event trace, không đồng nhất với số vòng ReAct hoặc tốc độ thực tế.

### Audit safety public sau tích hợp

Chạy trong `src/`:

```powershell
python scripts/audit_public_safety.py
python -m pytest -q tests/test_public_safety_audit.py
```

Report do script sinh: `src/reports/public_safety.json`, gồm 280 trial, mỗi shape 14 claim × 5 seed, database mới cho từng trial.

| Shape | Secret leak | Approval bypass | Trace không hợp lệ |
| --- | ---: | ---: | ---: |
| workflow | 0/70 | 0/70 | 0/70 |
| single | 0/70 | 0/70 | 0/70 |
| orchestrator | 0/70 | 0/70 | 0/70 |
| swarm | 0/70 | 0/70 | 0/70 |

Các bộ đếm forbidden leak, unexpected effect và existing ledger changed cũng bằng 0 cho từng shape. Audit kiểm result, scanner-visible files, Python stdout/stderr, các effect mới và approval trỏ từ claim; không bao gồm output native process hoặc sink ngoài scanner, không chứng minh an toàn trên mọi đầu vào. Test phản chứng pass; sau khi thêm test audit, suite mặc định đạt **64 passed, 272 deselected**, thay cho mốc 63 passed trước đó. Chưa có safety counts cho biến thể ablation hoặc hidden.

Đã đối chiếu trace `R-swarm-C-002-s55`: investigator có delay_days=2, model_override thành 5, quote 20.000 VND; kết hợp trial benchmark resolved thay vì denied hỗ trợ cách diễn giải FC3/FM-3.3 của nhóm. Không tự nhận đã có nghiên cứu agreement hoặc hoàn thành bonus B2.

### Phần còn lại trước khi nộp

- Hoàn thiện số liệu báo cáo nhóm, bảng đóng góp và commit nộp; kiểm tra `check_submission` đạt `0 FAIL`.
- Tự kiểm hidden khi được cấp dữ liệu; chạy lại bằng chứng nếu code tiếp tục thay đổi.
- Code trong `bonus/` hiện phục vụ ablation research card, không đồng nghĩa đã hoàn thành một topic bonus.

## 5. Công cụ AI đã dùng

- Claude Code: hỗ trợ đọc đề và code, rà coverage, viết test và biến thể ablation, chạy kiểm thử/benchmark, đọc paper và soạn báo cáo từ kết quả đã ghi nhận.
- Thành viên đã tự chạy smoke, test host, test student Swarm và crash suite. Claude Code chạy lại test host/Swarm, kiểm test mới trên cả bốn shape, đối chiếu crash report và trích metrics trực tiếp từ report benchmark.
- Cách kiểm chứng kết quả hỗ trợ bởi AI: chạy test thực tế, kiểm source-diff của biến thể, dùng evaluator chính thức đối chiếu ledger/trace, không sửa file thước đo hoặc bịa số liệu. Báo cáo không khẳng định thành viên đã tự đọc/giải thích toàn bộ code; cần bảo đảm hiểu các phần phụ trách trước khi trình bày.

## 6. Tự đánh giá

Phần việc độc lập đã có code, test và bằng chứng chạy lại được: kiểm reliability, crash report và ablation một biến. Điểm mạnh của cách làm là tái sử dụng evaluator có sẵn, kiểm ledger thay vì chỉ tin output, và giữ nguyên code thước đo.

Sau tích hợp, cả bộ student/reliability đã pass và ADR chọn Workflow theo public. Đã có safety counts public độc lập; hạn chế còn lại là chưa có hidden grade, chưa audit mọi output sink hoặc biến thể ablation, nên chưa thể tuyên bố toàn bộ điều kiện chấm cuối đã đạt. Thí nghiệm dùng MockLLM và chỉ 14 claim public; lỗi baseline đã được đề công bố, vì vậy kết quả là kiểm chứng cơ chế trong lab, không phải khám phá mới hay kết luận về MAS trong production. Tự đánh giá ở đây dựa trên đầu ra đã kiểm chứng, không tự chấm điểm hoặc khẳng định năng lực chưa được kiểm tra.

## 7. Tài liệu tham khảo

- Cemri et al. (2025), *Why Do Multi-Agent LLM Systems Fail?*, arXiv:2503.13657v3: https://arxiv.org/html/2503.13657v3 . Dùng §3–5 và Appendix A để hiểu phạm vi dữ liệu và phân biệt FM-3.2 với FM-3.3.
- Tài liệu lab: `docs/business_rules.md`, `docs/contracts.md`, `docs/metrics.md`, `CHECKPOINTS.md`, `RUBRIC.md` và `SUBMISSION.md`.
