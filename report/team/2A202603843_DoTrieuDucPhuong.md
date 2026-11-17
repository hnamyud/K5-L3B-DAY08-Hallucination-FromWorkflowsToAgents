# Báo cáo cá nhân — 2A202603843 - Đỗ Triệu Đức Phương

## 1. Thông tin thành viên

- Họ tên: Đỗ Triệu Đức Phương
- MSSV: 2A202603843
- Email: 26ai.phuongdtd@vinuni.edu.vn
- GitHub: ducphuong2005

## 2. Phần việc phụ trách

TV2 — triển khai shape B (single agent) trong `src/lab8/shapes/single.py` và viết test riêng trong `src/tests/test_single_agent.py`.

## 3. Đóng góp cụ thể

- Thay stub `not_implemented` bằng `run_claim` dùng `runtime.run_react` với giới hạn 8 bước. Agent lần lượt dùng các tool đọc `lookup_order`, `lookup_policy`, `lookup_refund_history` và `compute_compensation`; bước đọc claim ban đầu dùng `get_claim_view`.
- Dùng order có cấu trúc và policy hiện hành làm nguồn để tính quote. Không lấy số ngày trễ hay chỉ thị hoàn tiền từ câu chữ ticket. MockLLM được gọi trong mỗi lượt quyết định để ghi số lần gọi model, token và chi phí mô phỏng vào `Budget`; vì MockLLM chỉ trả chuỗi, thứ tự chọn tool hiện do hàm `decide` trong Python quy định.
- Chỉ code của shape gọi `durability.issue_refund` sau vòng ReAct. Truyền quote đầy đủ để host kiểm approval và trừ khoản hoàn trước đó theo đơn. Truyền tiếp `crash_at` và để `InjectedCrash` lan ra cho bài kiểm tra resume.
- Trả `RunResult` bằng `common.finish()` và ghi trace ở các nhánh thành công, từ chối, thiếu thông tin, escalated và lỗi. Trace chỉ ghi các trường đã chọn; lỗi model được ghi theo loại ngoại lệ, không ghi nội dung ngoại lệ có thể chứa dữ liệu nhạy cảm.
- Thêm hai test: `test_single_model_failure_is_safe_and_does_not_write` và `test_single_uses_bounded_read_tools_and_full_quote`.
- Commit cho phần việc này: feat: done single agent

## 4. Bằng chứng

- Mã nguồn: `src/lab8/shapes/single.py`; test: `src/tests/test_single_agent.py`.
- Chạy trong `src/`: `python -m pytest -q -m student -o addopts= -k 'single and not workflow and not orchestrator and not swarm'` → **62 passed**.
- Chạy trong `src/`: `python -m pytest -q` → **56 passed, 248 deselected**. Hai test mới nằm trong 56 test mặc định này.
- Test trường hợp C-014 xác nhận `proposal_amount = 150000` VND và refund thực ghi `120000` VND; agent chỉ chọn tool đọc và số lần gọi model không vượt 8.

## 5. Công cụ AI đã dùng

Đã dùng Codex để đọc yêu cầu, triển khai `single.py`, viết test và giải thích luồng xử lý. Kết quả được kiểm bằng bộ test student của riêng single agent, bộ pytest mặc định và `git diff --check`. 

## 6. Tự đánh giá

Phần single agent đã qua 62 test student public của shape B và các test mặc định tại thời điểm kiểm tra. Giới hạn hiện tại là MockLLM của starter không sinh tool call có cấu trúc, nên `decide` chọn tool theo quy tắc Python