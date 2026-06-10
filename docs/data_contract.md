# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| policy_refund_v4 | CSV Export | Sai cửa sổ hoàn tiền (14 ngày thay vì 7 ngày) | `no_stale_refund_window` / Halt |
| hr_leave_policy | CSV Export | Xung đột phiên bản (10 ngày phép năm của 2025 vs 12 ngày của 2026) | `hr_leave_no_stale_10d_annual` / Halt |
| access_control_sop | CSV Export | Tài liệu mới chưa được đăng ký trong pipeline | `unknown_doc_id` / Quarantine |
| sla_p1_2026 | CSV Export | Ngày hiệu lực (effective_date) sai định dạng | `invalid_effective_date_format` / Quarantine |
| it_helpdesk_faq | CSV Export | Thiếu ngày hiệu lực hoặc nội dung chunk trống | `missing_effective_date`, `missing_chunk_text` / Quarantine |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Khóa chính của chunk, được sinh từ hash `doc_id \| chunk_text \| seq` |
| doc_id | string | Có | Tên nguồn tài liệu canonical (e.g. `policy_refund_v4`, `access_control_sop`) |
| chunk_text | string | Có | Nội dung văn bản của chunk (phải có độ dài tối thiểu là 8 ký tự) |
| effective_date | date | Có | Ngày hiệu lực dưới định dạng ISO `YYYY-MM-DD` |
| exported_at | datetime | Có | Thời gian xuất bản dữ liệu từ hệ thống nguồn |

---

## 3. Quy tắc quarantine vs drop

*   **Quarantine (Cách ly):** Các bản ghi vi phạm quy tắc schema hoặc metadata (như sai định dạng ngày, thiếu nội dung, hoặc doc_id lạ không nằm trong allowlist) sẽ được chuyển vào thư mục `artifacts/quarantine/` dưới dạng file CSV để điều tra. Chúng không bị nạp vào vector store.
*   **Drop (Hủy):** Các bản ghi trùng lặp nội dung hoàn toàn (`duplicate_chunk_text`) chỉ giữ lại bản ghi đầu tiên, các bản ghi sau sẽ bị loại bỏ để giảm độ nhiễu.
*   **Approve / Merge:** Data Engineer hoặc Data Owner sau khi sửa lỗi dữ liệu thô (hoặc cập nhật schema/allowlist) sẽ chạy lại pipeline để tự động hòa trộn dữ liệu sạch.

---

## 4. Phiên bản & canonical

*   **Source of truth cho policy refund:** Tài liệu gốc là `data/docs/policy_refund_v4.txt`. Cửa sổ hoàn tiền chuẩn là **7 ngày làm việc**. Mọi chunk chứa "14 ngày làm việc" đều là dữ liệu cũ (stale) và tự động được sửa đổi thành "7 ngày làm việc" bởi pipeline.
*   **Source of truth cho HR leave policy:** Tài liệu gốc là `data/docs/hr_leave_policy.txt`. Nhân viên dưới 3 năm kinh nghiệm được hưởng **12 ngày phép năm** (theo chính sách 2026 áp dụng từ 2026-01-01). Mọi bản ghi phiên bản 2025 (10 ngày phép năm) hoặc có ngày hiệu lực trước 2026-01-01 đều là stale và bị đưa vào quarantine.
