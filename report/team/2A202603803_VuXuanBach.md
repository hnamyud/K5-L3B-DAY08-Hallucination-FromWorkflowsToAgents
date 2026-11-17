# Báo cáo cá nhân — 2A202603803_VuXuanBach

## 1. Thông tin thành viên

- Họ tên: Vũ Xuân Bách
- MSSV: 2A202603803
- Email: 26ai.bachvx2@vinuni.edu.vn
- GitHub: bachvx1709

## 2. Phần việc phụ trách

- Phụ trách chính Kiến trúc Shape C: **Orchestrator** (`src/lab8/shapes/orchestrator.py`).
- Xây dựng ranh giới bảo mật Handoff và điều tra viên chỉ đọc (Read-only Investigators).
- Viết bộ unit test chuyên biệt cho Orchestrator (`src/tests/test_orchestrator_student.py`).

## 3. Đóng góp cụ thể

- **Cài đặt Shape C (`src/lab8/shapes/orchestrator.py`):**
  - Triển khai cơ chế Handoff an toàn với `make_handoff`, chỉ cấp danh sách công cụ đọc (`lookup_order`, `lookup_policy`, `compute_compensation`) cho Investigator; tuyệt đối không cấp quyền ghi (`issue_refund`).
  - Đảm bảo ranh giới bảo mật: Chỉ trace gói handoff đã được sanitize, không bôi nháp context gốc hay claim row thô (tránh rò rỉ secret và PII).
  - Tuân thủ nguyên tắc tính tiền: Lấy số tiền bồi thường (`proposal`) trực tiếp từ `compute_compensation`, không để MockLLM ghi đè hoặc tự tính toán.
  - Xử lý Crash Safety: Chuyển tiếp tham số `crash_at` vào `durability.issue_refund`, bắt `InjectedCrash` để kịp xả trace (`trace.write()`) rồi re-raise ngoại lệ ra ngoài theo đúng chuẩn test T12.
  - Luôn truyền quote đầy đủ vào `issue_refund` để Host tự tính trần và trừ các khoản đã hoàn trước đó (đáp ứng đúng T13, T14).
- **Thêm bài kiểm thử (`src/tests/test_orchestrator_student.py`):**
  - Viết 4 bài test độc lập kiểm tra: xử lý claim không tồn tại (`unknown_claim`), xử lý claim thiếu mã đơn (`needs_info`), kiểm tra chống rò rỉ token `CANARY_SECRET_*`, và kiểm tra xử lý từ chối khi đơn hàng không thuộc sở hữu của principal (`escalated`).

## 4. Bằng chứng

- Bộ test Student Matrix: Vượt qua **62/62 tests** cho orchestrator (`python -m pytest -q -m student -o addopts= -k orchestrator`).
- Unit test cá nhân: Pass **4/4 tests** trong `src/tests/test_orchestrator_student.py`.
- Crash safety & Idempotency: Xanh test T11 (gọi lại cùng claim vẫn 1 dòng ledger) và T12 (crash giữa chừng rồi resume vẫn chỉ 1 refund).
- Test file: `src/tests/test_orchestrator_student.py`.

## 5. Công cụ AI đã dùng

- **Công cụ:** Antigravity AI Assistant.
- **Mục đích sử dụng:**
  - Đọc `README.md`, `RULES.md`, `CHECKPOINTS.md` để tổng hợp thông tin bài lab.
  - Hỗ trợ xây dựng khung code mẫu cho `orchestrator.py` tuân thủ strict contract.
  - Gợi ý các kịch bản kiểm thử biên (edge cases) và cấu trúc test case cho Pytest.
- **Cách tự kiểm tra kết quả:**
  - Chạy `pytest` xác minh trực tiếp toàn bộ 62 test của student suite và 4 ca test mới viết.
  - Kiểm tra thủ công nội dung file trace sinh ra để đảm bảo không rò rỉ chuỗi canary hay thông tin cá nhân.

## 6. Tự đánh giá

- Hoàn thành đầy đủ các yêu cầu kỹ thuật của Shape C.
- Đảm bảo 100% các điều kiện Must-Pass liên quan đến ranh giới quyền lực (không bypass approval), an toàn dữ liệu (không leak secret) và an toàn khi crash (idempotency).
- Hoàn thành trọn vẹn phần việc được phân công.
