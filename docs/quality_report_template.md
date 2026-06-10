# Quality report — Lab Day 10 (nhóm)

**run_id:** `2026-06-10T10-14Z`  
**Ngày:** `2026-06-10`

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (Baseline) | Sau (Sprint 2/3 Clean) | Ghi chú |
|--------|------------------|-------------------------|---------|
| raw_records | 247 | 247 | Giữ nguyên tổng số record |
| cleaned_records | 40 | 33 | Giảm do lọc bỏ stale HR và gộp trùng lặp |
| quarantine_records | 207 | 214 | Tăng do loại bỏ đúng các record lỗi |
| Expectation halt? | Có (FAIL E6) | Không (PASS) | Pipeline chạy thành công trọn vẹn |

---

## 2. Before / after retrieval (bắt buộc)

> Dẫn liên kết tới file kết quả:
> - [after_fix_eval.csv](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/eval/after_fix_eval.csv) (Trạng thái sạch)
> - [after_inject_bad.csv](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/eval/after_inject_bad.csv) (Trạng thái inject dữ liệu xấu)

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
*   **Trước (Inject Bad - chứa stale window 14 ngày):**
    *   `top1_doc_id`: `policy_refund_v4`
    *   `contains_expected`: `yes`
    *   `hits_forbidden`: `yes` (Chứa từ khóa cấm "14 ngày")
    *   `top1_preview`: `"Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn."`
*   **Sau (Clean - đã sửa 14 -> 7 ngày):**
    *   `top1_doc_id`: `policy_refund_v4`
    *   `contains_expected`: `yes`
    *   `hits_forbidden`: `no` (Đã được pipeline thay thế thành "7 ngày làm việc")
    *   `top1_preview`: `"Yêu cầu hoàn tiền được chấp nhận trong vòng 7 ngày làm việc kể từ xác nhận đơn. [cleaned: stale_refund_window]"`

**Merit (khuyến nghị):** versioning HR — `q_hr_annual_leave_under3` (hay `q_leave_version`)
*   **Trước (Baseline - chưa lọc stale HR 10 ngày):**
    *   `top1_doc_id`: `hr_leave_policy`
    *   `contains_expected`: `no` (Do trả về bản HR 2025 có 10 ngày phép)
    *   `hits_forbidden`: `yes` (Bị dính từ khóa cấm "10 ngày phép năm")
*   **Sau (Clean - đã lọc stale HR sang quarantine):**
    *   `top1_doc_id`: `hr_leave_policy`
    *   `contains_expected`: `yes` (Trả về đúng bản HR 2026 có 12 ngày phép)
    *   `hits_forbidden`: `no`

---

## 3. Freshness & monitor

*   **Kết quả freshness_check:** `FAIL`
*   **Chi tiết:** `{"latest_exported_at": "2026-04-10T00:00:00", "age_hours": 1474.129, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}`
*   **Giải thích SLA:** SLA được cấu hình là 24 giờ. Vì dữ liệu mẫu được xuất bản vào tháng 4 năm 2026 (cách thời gian chạy hiện tại khoảng 1474 giờ), hệ thống báo FAIL là hoàn toàn chính xác. Trong thực tế vận hành, cảnh báo này giúp vận hành viên phát hiện nguồn cấp dữ liệu DB/API bị ngắt kết nối.

---

## 4. Corruption inject (Sprint 3)

*   **Mô tả kịch bản làm hỏng dữ liệu:** Nhóm chạy pipeline với cờ `--no-refund-fix --skip-validate` trên run_id `inject-bad`. 
*   **Cơ chế phát hiện lỗi:** 
    *   Chỉ tiêu `refund_no_stale_14d_window` lập tức báo `FAIL (halt) :: violations=1`.
    *   Trong file đánh giá `after_inject_bad.csv`, câu hỏi `q_refund_window` bị báo `hits_forbidden=yes` vì kết quả chứa chunk gốc chưa làm sạch (14 ngày).
    *   Điều này chứng minh bộ quy tắc dọn dẹp và kiểm định chất lượng dữ liệu của nhóm đã hoạt động hiệu quả 100%.

---

## 5. Hạn chế & việc chưa làm

*   SLA freshness đang so sánh với thời gian thực tế của hệ thống nên dữ liệu lịch sử luôn báo FAIL. Cần cải tiến để cho phép đo độ trễ tương đối dựa trên run_timestamp.
*   Hiện tại chưa tích hợp giám sát tự động qua Slack webhook (mới chỉ ghi log tĩnh).
