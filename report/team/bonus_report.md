# Bonus report: B4 · Baseline công bằng, ablation và thống kê

- **Topic + nguồn :** B4 · Baseline công bằng, ablation và thống kê — Nguồn: EMNLP 2026 (*Rethinking Evaluation of MAS Efficiency*, slide 44, 59) và AgentArch (AACL 2026, slide 43).
- **CLAIM của nguồn:** "Efficiency gain" và tính ưu việt của hệ thống Multi-Agent thường bị thổi phồng do thiết kế benchmark thiếu baseline kiểm chứng công bằng và thiếu kiểm định thống kê nghiêm ngặt.
- **BOUNDARY của nguồn:** Nghiên cứu chỉ so sánh trên các bài toán có thể đo lường và xác minh bằng dữ liệu cấu trúc; không áp dụng cho các tác vụ sáng tạo tự do hoặc không có ground truth.

---

## 1. Phương pháp và cài đặt

1. Ablation baseline (`src/bonus/swarm_no_override.py`):
   - Tắt `model_override`, cơ chế khiến Swarm dùng số ngày trễ trong ticket thay cho dữ kiện đã tra cứu từ cơ sở dữ liệu khi seed % 5 == 0.
   - Đo lại hiệu năng của Swarm sau khi tắt cơ chế này.
2. Kiểm định thống kê (`src/bonus/stats.py`):
   - Tính Bootstrap 95% Confidence Interval cho Pass Rate của từng shape bằng cách resampling 1000 lần ở cấp độ Claim.
   - Dùng kiểm định bắt cặp McNemar (Exact McNemar Test) trên toàn bộ 70 trials (14 claims × 5 seeds) để so sánh baseline `workflow` với `swarm`.
3. Mã nguồn và kiểm thử:
   - Script chạy thống kê: `src/bonus/stats.py`.
   - Bộ unit tests: `src/tests/test_bonus_ablation_stats.py` và `src/tests/test_bonus_swarm_no_override.py`.
   - Dữ liệu kết quả: `src/reports/bonus_stats.json` và `src/reports/bonus_ablation.json`.

---

## 2. Kết quả

### Bảng 1: So sánh hiệu năng và Khoảng tin cậy Bootstrap 95% CI (14 claims × 5 seeds = 70 trials)

| Kiến trúc (Shape) | Pass Rate Thực nghiệm | 95% Bootstrap CI | Pass^5 | Variance | Cost/Success | Avg Steps |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **workflow** (Nhóm chọn) | **1.0000** | **[1.0000, 1.0000]** | **1.0000** | **0.000000** | **27.86** | **1.00** |
| **single** | 1.0000 | [1.0000, 1.0000] | 1.0000 | 0.000000 | 46.80 | 9.64 |
| **orchestrator** | 1.0000 | [1.0000, 1.0000] | 1.0000 | 0.000000 | 47.85 | 2.79 |
| **swarm** (Reference) | 0.9857 | [0.9571, 1.0000] | 0.9286 | 0.000816 | 49.57 | 2.80 |
| **swarm_no_override** (Ablation) | 1.0000 | [1.0000, 1.0000] | 1.0000 | 0.000000 | 49.12 | 2.79 |

### Bảng 2: Kết quả kiểm định bắt cặp McNemar Test (so với Workflow)

| Cặp so sánh | Số trial bất đồng (Discordant pairs) | Chi tiết lệch ($b / c$) | $p$-value (Exact) | Ý nghĩa thống kê ($\alpha=0.05$) |
|---|:---:|:---:|:---:|:---:|
| **workflow vs swarm** | 1 | $b=1$ (Workflow pass, Swarm fail) / $c=0$ | $p = 1.0000$ | Không đủ bác bỏ $H_0$ trên $N=70$ |
| **workflow vs single** | 0 | $b=0 / c=0$ | $p = 1.0000$ | Giống hệt nhau về kết quả |
| **workflow vs orchestrator** | 0 | $b=0 / c=0$ | $p = 1.0000$ | Giống hệt nhau về kết quả |

---

## 3. Phân tích kết quả

### Q1: Sự chênh lệch giữa shape nhóm chọn (Workflow) và Swarm có ý nghĩa thống kê không?
- Trên tập public ($N=70$ trials), Swarm fail 1 lần ở claim C-002 seed 55 do ghi đè dữ kiện bằng số ngày trễ trong ticket. Với $N=70$ và số cặp bất đồng $n=1$, kiểm định McNemar chính xác cho $p = 1.0000 > 0.05$. Chênh lệch Pass Rate trên mẫu này chưa đạt mức ý nghĩa thống kê $p < 0.05$.
- Workflow có Cost/Success là 27.86/success, thấp hơn 43.8% so với Swarm (49.57), và Variance bằng 0.000000. Trên task này, Multi-Agent tốn thêm chi phí và có phương sai cao hơn mà chưa cải thiện độ đúng so với Workflow. Kết quả phù hợp với mối quan tâm của EMNLP 2026 về cách đánh giá hiệu năng MAS; riêng thí nghiệm này chưa đủ để chứng minh kết luận của nguồn cho các task khác.

### Q2: Thành phần nào bỏ đi mà metric không đổi?
- Khi tắt `model_override` trong `swarm_no_override`, Pass Rate của Swarm tăng từ $0.9857 \rightarrow 1.0000$ và Pass^5 từ $0.9286 \rightarrow 1.0000$.
- Swarm vẫn giữ các vòng peer communication và có chi phí cao hơn Workflow, nhưng độ đúng chưa cao hơn trên tập đã đo. Ablation này chưa tắt riêng peer communication nên chưa xác định được ảnh hưởng độc lập của thành phần đó.

### Q3: Giới hạn của nghiên cứu (Limitations)
1. Hệ thống dùng MockLLM với seed cố định. Trên LLM thật, phương sai và xác suất ảo giác của Swarm có thể lớn hơn; thí nghiệm hiện tại chưa đo được khả năng này.
2. LogiX-VN có dữ liệu rõ ràng và luật nghiệp vụ xác định (deterministic business logic). Với bài toán mở, ngữ cảnh mơ hồ hoặc đầu vào đa phương thức (multimodal), Agent có thể xử lý được những yêu cầu vượt ngoài Workflow cố định. Cần thử nghiệm riêng để kiểm tra nhận định đó.

---

## Điều kiện xét điểm bonus

Nhóm chọn Workflow làm kiến trúc chính. Sau khi tích hợp bonus, suite mặc định đạt **68 passed**, suite student/reliability đạt **272 passed** và crash suite đạt **20/20**. Audit public của Workflow trên 70 trial ghi nhận **secret leak = 0** và **approval bypass = 0**, theo `src/reports/public_safety.json`.

Self-check với `student_drop` cho Workflow chạy hết **200 trial (40 claim × 5 seed)**, có **0 lỗi chạy**, **0 leak suspect** và không báo lỗi trace. Kết quả lưu trong `src/reports/hidden_selfcheck.json`. Tỉ lệ trial chạy không lỗi là **200/200 = 100%**; dữ liệu không có nhãn nên chưa xác định được tỉ lệ trial đúng nghiệp vụ, hidden pass rate vẫn là `null`.

Ngưỡng **hidden pass rate ≥ 0.80** và ba must-pass trên hidden chờ grader của giảng viên xác nhận. Các kết quả public, crash suite và self-check chưa đủ để kết luận nhóm đã qua gate bonus. Điểm bonus còn phụ thuộc đánh giá ba tiêu chí: chạy được, đo công bằng và phân tích có giới hạn; nhóm chưa tự kết luận được nhận đủ +10.

## 4. Lệnh tái lập kết quả

Chạy trong thư mục `src/`:
```bash
# 1. Chạy phân tích thống kê toán học
python -m bonus.stats --in reports/benchmark.json --out reports/bonus_stats.json

# 2. Chạy toàn bộ test kiểm thử của Topic B4
pytest -q tests/test_bonus_ablation_stats.py tests/test_bonus_swarm_no_override.py
```
