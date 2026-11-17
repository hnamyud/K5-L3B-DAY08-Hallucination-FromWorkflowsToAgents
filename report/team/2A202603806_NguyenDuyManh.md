# Báo cáo cá nhân — 2A202603806_NguyenDuyManh

Báo cáo phần Reliability & Evaluation đã thực hiện trên nhánh `feature/evaluation`. Benchmark chính thức bốn shape và ADR cuối sẽ được cập nhật sau tích hợp; các kết quả dưới đây không đại diện cho bản nộp cuối của cả nhóm.

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
- Commit kỹ thuật: `8778b03` — `test: add reliability checks and controlled Swarm ablation`.

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

### Phần còn phụ thuộc tích hợp

- Chạy toàn bộ student/reliability tests khi ba shape không còn stub.
- Chạy lại crash suite và benchmark chính thức bốn shape để tạo `src/reports/benchmark.json`; ablation report không thay thế file này.
- Chốt ADR, số liệu báo cáo nhóm, commit nộp và kiểm tra `check_submission` đạt `0 FAIL`.
- Code trong `bonus/` hiện phục vụ ablation research card, không đồng nghĩa đã hoàn thành một topic bonus.

## 5. Công cụ AI đã dùng

- Claude Code: hỗ trợ đọc đề và code, rà coverage, viết test và biến thể ablation, chạy kiểm thử/benchmark, đọc paper và soạn báo cáo từ kết quả đã ghi nhận.
- Thành viên đã tự chạy smoke, test host, test student Swarm và crash suite. Claude Code chạy lại test host/Swarm, kiểm test mới trên cả bốn shape, đối chiếu crash report và trích metrics trực tiếp từ report benchmark.
- Cách kiểm chứng kết quả hỗ trợ bởi AI: chạy test thực tế, kiểm source-diff của biến thể, dùng evaluator chính thức đối chiếu ledger/trace, không sửa file thước đo hoặc bịa số liệu. Báo cáo không khẳng định thành viên đã tự đọc/giải thích toàn bộ code; cần bảo đảm hiểu các phần phụ trách trước khi trình bày.

## 6. Tự đánh giá

Phần việc độc lập đã có code, test và bằng chứng chạy lại được: kiểm reliability, crash report và ablation một biến. Điểm mạnh của cách làm là tái sử dụng evaluator có sẵn, kiểm ledger thay vì chỉ tin output, và giữ nguyên code thước đo.

Hạn chế hiện tại là ba shape chưa tích hợp trong các lần kiểm trên, nên chưa đánh giá được độ an toàn hoặc chọn kiến trúc cuối cho nhóm. Thí nghiệm dùng MockLLM và chỉ 14 claim public; lỗi baseline đã được đề công bố, vì vậy kết quả là kiểm chứng cơ chế trong lab, không phải khám phá mới hay kết luận về MAS trong production. Tự đánh giá ở đây dựa trên đầu ra đã kiểm chứng, không tự chấm điểm hoặc khẳng định năng lực chưa được kiểm tra.

## 7. Tài liệu tham khảo

- Cemri et al. (2025), *Why Do Multi-Agent LLM Systems Fail?*, arXiv:2503.13657v3: https://arxiv.org/html/2503.13657v3 . Dùng §3–5 và Appendix A để hiểu phạm vi dữ liệu và phân biệt FM-3.2 với FM-3.3.
- Tài liệu lab: `docs/business_rules.md`, `docs/contracts.md`, `docs/metrics.md`, `CHECKPOINTS.md`, `RUBRIC.md` và `SUBMISSION.md`.
