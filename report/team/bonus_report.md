# Bonus report: B4 · Baseline công bằng, ablation và thống kê

- **Topic + nguồn :** B4 · Baseline công bằng, ablation và thống kê — Nguồn: EMNLP 2026 (*Rethinking Evaluation of MAS Efficiency*, slide 44, 59) và AgentArch (AACL 2026, slide 43).
- **CLAIM của nguồn:** "Efficiency gain" và tính ưu việt của hệ thống Multi-Agent thường bị thổi phồng do thiết kế benchmark thiếu baseline kiểm chứng công bằng và thiếu kiểm định thống kê nghiêm ngặt.
- **BOUNDARY của nguồn:** Nghiên cứu chỉ so sánh trên các bài toán có thể đo lường và xác minh bằng dữ liệu cấu trúc; không áp dụng cho các tác vụ sáng tạo tự do hoặc không có ground truth.

---

## 1. Phương pháp & Cài đặt

1. **Ablation baseline (`src/bonus/swarm_no_override.py`):**
   - Loại bỏ thành phần `model_override` trong Swarm (vốn khiến Swarm tự ý tin tưởng số ngày trễ người dùng khai trong ticket thay vì tra cứu cơ sở dữ liệu ở seed % 5 == 0).
   - Kiểm tra xem khi loại bỏ hành vi này, hiệu năng của Swarm thay đổi ra sao.
2. **Kiểm định thống kê toán học (`src/bonus/stats.py`):**
   - Áp dụng phương pháp **Bootstrap 95% Confidence Interval** (resampling 1000 lần ở cấp độ Claim) để xác định khoảng tin cậy của Pass Rate cho từng shape.
   - Áp dụng **Kiểm định bắt cặp McNemar (Exact McNemar Test)** trên toàn bộ 70 trials (14 claims × 5 seeds) giữa baseline `workflow` và `swarm`.
3. **Mã nguồn và kiểm thử:**
   - Script chạy thống kê: `src/bonus/stats.py`.
   - Bộ unit tests: `src/tests/test_bonus_ablation_stats.py` và `src/tests/test_bonus_swarm_no_override.py`.
   - Dữ liệu kết quả: `src/reports/bonus_stats.json` và `src/reports/bonus_ablation.json`.

---

## 2. Kết quả đo lường định lượng

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

## 3. Phân tích & Trả lời câu hỏi nghiên cứu

### Q1: Sự chênh lệch giữa shape nhóm chọn (Workflow) và Swarm có ý nghĩa thống kê không?
- **Về Pass Rate:** Trên tập mẫu public ($N=70$ trials), Swarm chỉ rớt đúng 1 lần (tại claim C-002 seed 55 do ảo giác override). Với $N=70$ và số cặp bất đồng $n=1$, kiểm định McNemar chính xác cho $p = 1.0000 > 0.05$. Điều này cho thấy với kích thước mẫu nhỏ, sự khác biệt về Pass Rate chưa đạt mức ý nghĩa thống kê $p < 0.05$.
- **Về Chi phí (Cost/Success) & Độ ổn định (Variance):** Sự chênh lệch là **vượt trội và có ý nghĩa thực tế to lớn**. Workflow đạt chi phí chỉ **27.86/success**, rẻ hơn **43.8%** so với Swarm (49.57), trong khi phương sai bằng tuyệt đối 0.000000. Điều này chứng minh luận điểm của EMNLP 2026: Multi-Agent không mang lại lợi ích hiệu năng vượt trội trên task này, mà chỉ làm tăng chi phí và rủi ro phương sai.

### Q2: Thành phần nào bỏ đi mà metric không đổi?
- Kết quả ablation của `swarm_no_override` cho thấy: Việc loại bỏ cơ chế `model_override` đưa Pass Rate của Swarm từ $0.9857 \rightarrow 1.0000$ và Pass^5 từ $0.9286 \rightarrow 1.0000$. 
- Ngược lại, việc giữ nguyên các vòng lặp peer communication phức tạp trong Swarm chỉ làm tốn thêm chi phí mà không đem lại bất kỳ độ chính xác bổ sung nào so với Workflow.

### Q3: Giới hạn của nghiên cứu (Limitations)
1. **Mô hình MockLLM:** Hệ thống sử dụng MockLLM với seed cố định. Trên LLM thật, phương sai và xác suất ảo giác của Swarm có thể còn lớn hơn nữa.
2. **Loại hình bài toán:** LogiX-VN là bài toán nghiệp vụ có cấu trúc dữ liệu rõ ràng (deterministic business logic). Nếu bài toán chuyển sang dạng mở, suy luận ngữ cảnh mơ hồ hoặc đa phương thức (multimodal), kiến trúc Agent có thể phát huy tác dụng mà Workflow cứng không thể đáp ứng.

---

## 4. Lệnh tái lập kết quả

Chạy trong thư mục `src/`:
```bash
# 1. Chạy phân tích thống kê toán học
python -m bonus.stats --in reports/benchmark.json --out reports/bonus_stats.json

# 2. Chạy toàn bộ test kiểm thử của Topic B4
pytest -q tests/test_bonus_ablation_stats.py tests/test_bonus_swarm_no_override.py
```
