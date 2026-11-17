# Research card

File này nằm cố định ở `report/team/research_card.md`; `python -m lab8.check_submission` đọc cột "Nhóm điền" bằng máy.

Phần này nối tiếp phần mini research ở buổi lý thuyết (slide 62–63). Chọn **một** câu R1–R4, điền 6 dòng, rồi biến nó thành **một ablation** chạy được trong lab: giữ nguyên model (MockLLM), tool, seed, corpus và giá, chỉ thay đổi một biến.

| Câu | Nguồn (thư viện tham khảo, slide 68–69) | Câu hỏi gợi ý cho lab |
| --- | --- | --- |
| R1 · Scaling | Google Research 2026, *Towards a Science of Scaling Agent Systems* | Task claims chủ yếu tuần tự hay song song được? Dự đoán thêm investigator sẽ làm pass rate và cost/success thay đổi thế nào. |
| R2 · Failure taxonomy | MAST, NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* | Các trial fail của swarm (seed 55) thuộc nhóm nào: lỗi giao việc, lệch mục tiêu, hay lỗi kiểm chứng? |
| R3 · Architecture choices | AgentArch, AACL 2026 | Nếu đổi topology mà giữ nguyên mọi thứ khác, biến nào phải cố định để kết luận còn đúng? |
| R4 · Evaluation confounders | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* | Làm sao chứng minh chênh lệch giữa A/B/C/D là do topology, không phải do prompt, tool hay seed? |

## Phiếu của nhóm

Điền vào cột cuối. Không xóa hai cột đầu.

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R2 — Failure taxonomy: lỗi Swarm khi tin ticket thay vì dữ kiện order thuộc nhóm nào và có giảm khi bỏ cơ chế ghi đè không? |
| SOURCE | Paper/blog, năm, URL, loại bằng chứng (benchmark, production report, blog kinh nghiệm) | Cemri et al. (2025), *Why Do Multi-Agent LLM Systems Fail?*, arXiv:2503.13657v3 (26/10/2025). PDF: https://arxiv.org/pdf/2503.13657v3 ; HTML: https://arxiv.org/html/2503.13657v3 . Bằng chứng: nghiên cứu thực nghiệm trace, gán nhãn chuyên gia và LLM annotator, kèm intervention case studies; xem §3–5 và Appendix A. |
| CLAIM | Một kết luận gốc của nguồn. Không viết rộng hơn phạm vi paper | MAST xác định 14 failure modes thuộc ba nhóm: System Design Issues, Inter-Agent Misalignment và Task Verification. Theo §4 (FC3), có verifier không bảo đảm đầu ra đúng nếu chỉ kiểm tra hời hợt. Appendix A định nghĩa FM-3.2 là bỏ sót toàn bộ/một phần việc kiểm chứng; FM-3.3 là kiểm chứng hoặc đối chiếu không đầy đủ thông tin/quyết định quan trọng trong quá trình thực hiện. Đây là phân biệt để chẩn đoán lỗi, không phải bằng chứng rằng thêm agent luôn tốt hơn hoặc xấu hơn. |
| BOUNDARY | Model / dataset / setup giới hạn kết luận đó | Taxonomy ban đầu hình thành từ 150 trace của 5 framework, được 6 chuyên gia phân tích; độ đồng thuận gán nhãn sau tinh chỉnh κ=0,88 (§3.1–3.2). MAST-Data mở rộng gồm hơn 1.600 trace từ 7 framework. Paper lưu ý phân bố lỗi của các hệ thống chạy task/benchmark khác nhau không phải bảng xếp hạng hiệu năng trực tiếp (§5.1, Figure 4). Nhãn có thể chồng lấn và hành vi bề mặt giống nhau có thể có nguyên nhân khác (§4); không coi κ là độ chính xác phân loại cho lab. Thí nghiệm ở đây chỉ dùng MockLLM, corpus synthetic public và seed 11,22,33,44,55, không suy rộng mức cải thiện sang model thật. |
| HYPOTHESIS | Nếu đổi shape A→B (hoặc bật/tắt một thành phần), metric X sẽ đổi theo hướng nào, khoảng bao nhiêu | Đề xuất: nếu chỉ tắt việc Swarm ghi đè delay_days từ ticket lên dữ kiện order, trial C-002 seed 55 chuyển từ fail sang pass, pass^5 của C-002 từ 0 lên 1. Đây là dự đoán từ điểm yếu được công bố trong đề, chưa phải kết quả đo hoặc phát hiện mới. |
| METHOD | Biến giữ cố định: model, tools, seed, corpus, giá, evaluator. Biến duy nhất được thay đổi: … | Đã chạy Swarm gốc so với bonus.swarm_no_override, bản sao chỉ vô hiệu hóa điều kiện model_override và đổi tên shape để tách trace. Test source-diff kiểm _run chỉ có hai thay đổi đó. Giữ nguyên MockLLM, template prompt và cách gọi model, bộ tool, cách tính budget, 14 claim public, năm seed, pricing và evaluator; mỗi trial dùng database mới. Không sửa swarm_reference.py hoặc file thước đo. Số model calls giữ nguyên; nội dung draft prompt, nhánh tính lại quote, số tool calls và cost có thể đổi như hệ quả của can thiệp, không được coi là cố định. Benchmark A/B/C/D phục vụ chọn kiến trúc, không tự coi là ablation một biến vì prompt và luồng gọi tool có thể khác. |
| FAILURE / METRIC | Một lỗi dự kiến và cách đo nó: pass rate, pass^5, variance, cost/success, leak, bypass | Nhãn dự kiến: FC3 / FM-3.3 (Incorrect verification), vì Swarm đã có kết quả investigator từ order nhưng ghi đè delay_days theo ticket khi seed chia hết cho 5; cần kiểm trace C-002 seed 55 để xác nhận chuỗi handoff → investigator → model_override → quote và đối chiếu ledger. Đây là ánh xạ do nhóm đề xuất từ định nghĩa Appendix A, không phải nhãn do tác giả paper gán; không tự kết luận lỗi giao việc hoặc lệch mục tiêu chỉ từ trial fail. Nếu trace cho thấy thiếu hẳn bước kiểm chứng, cân nhắc FM-3.2 và ghi rõ tiêu chí. Đo bằng evaluator cố định: pass rate, pass^5, seed min–max/variance, cost/success; kiểm secret leak và approval bypass bằng 0. |

## Kết quả ablation

- Lệnh benchmark đã chạy (argv; chạy trong `src/`): `python -m lab8.benchmark --shapes swarm,swarm_no_override --extra-shape swarm_no_override=bonus.swarm_no_override:run_claim --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/bonus_ablation.json`. Trong phiên công cụ, lệnh được gọi qua `runpy` sau khi đổi cwd sang `src/`, với các đối số trên.
- Run id / file report: `src/reports/bonus_ablation.json`; cặp dẫn chứng `R-swarm-C-002-s55` và `R-swarm_no_override-C-002-s55`. Report chứa run_id từng trial, không có run_id chung.
- Kết quả so với hypothesis: được ủng hộ trên public. Mỗi biến thể có 70 trial (14 claim × 5 seed). Baseline pass 69/70; biến thể 70/70. Chỉ cặp C-002 seed 55 chuyển fail → pass (resolved → denied); 69 cặp còn lại giữ nguyên pass. Không có trial mới bị fail; pass^5 riêng C-002 từ 0 lên 1. Chưa thực hiện kiểm định thống kê hoặc hidden cho biến thể.

| Metric (từ report sinh tự động) | Swarm gốc | Tắt model_override |
| --- | ---: | ---: |
| Pass rate | 0.985714 | 1.000000 |
| Pass^5 | 0.928571 | 1.000000 |
| Seed min–max | 0.928571–1.000000 | 1.000000–1.000000 |
| Seed variance | 0.000816327 | 0 |
| Cost/success (VND mô phỏng) | 49.566957 | 48.715714 |
| Tổng cost (VND mô phỏng) | 3420.12 | 3410.10 |
| Mean / p95 steps | 2.8 / 3 | 2.785714 / 3 |
| Mean model calls | 3 | 3 |
| Mean tool calls | 4.657143 | 4.642857 |

Cost giảm không chỉ vì thêm một success vào mẫu số: trial C-002 seed 55 còn bỏ một lần tính lại quote, tool calls 6 → 5, cost 62.22 → 52.20 VND mô phỏng. Report benchmark không có safety counts riêng; không suy diễn secret leak/approval bypass bằng 0 chỉ từ pass rate hoặc từ crash suite. Cần audit safety riêng nếu báo hai số này.

- Giới hạn: MockLLM không phải model sản xuất, latency là `unavailable`. Điểm yếu Swarm đã được đề công bố; ablation chỉ kiểm chứng cơ chế cụ thể trong lab, không tái lập toàn bộ nghiên cứu MAST và không chứng minh topology Swarm tự thân gây lỗi.

Muốn đi xa hơn một ablation để lấy điểm cộng thì xem [docs/bonus_topics.md](../../docs/bonus_topics.md).
