# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** AI-Action-Group-1  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Tiến Đạt (MSHV: 2A202600595) | Ingestion / Cleaning Owner | dat.nt@company.internal |
| Nhóm viên 2 | Quality Owner | quality@company.internal |
| Nhóm viên 3 | Embed Owner | embed@company.internal |
| Nhóm viên 4 | Monitor Owner | monitor@company.internal |

**Ngày nộp:** 2026-06-10  
**Repo:** VinUni-AI20k/Lecture-Day-08-09-10  
**Độ dài khuyến nghị:** 600–1000 từ

---

## 1. Pipeline tổng quan (150–200 từ)

Nguồn dữ liệu thô (raw) được lấy từ file CSV thô [policy_export_dirty.csv](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/data/raw/policy_export_dirty.csv). File này chứa các dữ liệu thô xuất từ 5 hệ thống nguồn khác nhau: CS refund policy, SLA P1, IT helpdesk FAQ, HR leave policy và quy trình cấp quyền Admin access.

Chuỗi lệnh chạy end-to-end cho pipeline là:
`python etl_pipeline.py run`

Mỗi lần chạy sẽ sinh ra một `run_id` độc nhất dưới dạng timestamp (ví dụ: `2026-06-10T09-26Z`). `run_id` này được ghi nhận trực tiếp ở dòng đầu tiên của file log trong thư mục `artifacts/logs/` và trong manifest tương ứng tại `artifacts/manifests/`.

---

## 2. Cleaning & expectation (150–200 từ)

Nhóm đã bổ sung thêm 5 quy tắc dọn dẹp (cleaning rules) và 3 chỉ tiêu kiểm định chất lượng (expectations) mới.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `access_control_sop` allowlist | 0 cleaned records | 6 cleaned records | `quarantine` giảm từ 207 xuống 201 |
| `stale_hr_policy_text` (Quarantine) | 2 violations (FAIL E6) | 0 violations (PASS E6) | `quarantine` tăng từ 201 lên 203 |
| `no_exclamation_markers` (E7 / Cleaning) | 3 records có `!!!` | 0 records có `!!!` | PASS expectation `no_exclamation_markers` |
| `no_repeated_working_days` (E8 / Cleaning) | 3 records lặp từ | 0 records lặp từ | PASS expectation `no_repeated_working_days` |
| `exported_at_iso_format` (E9 / Cleaning) | 17 records sai format | 0 records sai format | PASS expectation `exported_at_iso_format` |

**Rule chính (baseline + mở rộng):**
- Bổ sung `access_control_sop` vào allowlist để đưa quy trình Admin access vào KB.
- Lọc bỏ phiên bản phép năm cũ (10 ngày phép năm của 2025) sang quarantine.
- Loại bỏ các ký tự dấu than `!!!` thừa trong chunk text.
- Loại bỏ các tiền tố boilerplate như `Nội dung không rõ ràng:` và các ghi chú hệ thống.
- Chuẩn hóa lặp từ "làm việc làm việc".
- Chuẩn hóa định dạng ngày xuất bản `exported_at` (thay thế `/` thành `-`).

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**
Khi chạy baseline, pipeline đã bị dừng lại (halt) tại expectation `hr_leave_no_stale_10d_annual` do có 2 bản ghi phép năm cũ. Nhóm đã xử lý bằng cách thêm quy tắc lọc bỏ bản ghi phép năm cũ (chứa `"10 ngày phép năm"`) và đưa chúng vào quarantine.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

**Kịch bản inject:**
Trong Sprint 3, nhóm đã cố ý làm hỏng dữ liệu thông qua cờ `--no-refund-fix --skip-validate`. Kịch bản này bỏ qua quy tắc thay thế cửa sổ hoàn tiền của `policy_refund_v4` (giữ nguyên lỗi 14 ngày phép cũ) và bỏ qua việc dừng pipeline khi phát hiện lỗi dữ liệu để nạp trực tiếp vào vector database dưới `run_id` `inject-bad`.

**Kết quả định lượng:**
*   **Khi chạy mô phỏng lỗi (Inject Bad):** Dữ liệu hoàn tiền thô (14 ngày làm việc) được lưu thẳng vào ChromaDB. Khi chạy đánh giá RAG [after_inject_bad.csv](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/eval/after_inject_bad.csv), câu hỏi `q_refund_window` trả về `hits_forbidden=yes` do văn bản tìm được chứa từ khóa cấm "14 ngày".
*   **Khi chạy chuẩn (Clean State):** Pipeline sửa đổi 14 ngày thành 7 ngày, kết quả trong [after_fix_eval.csv](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/eval/after_fix_eval.csv) trả về đúng và sạch `hits_forbidden=no`.
*   **Độ chính xác chấm điểm:** Đạt 10/10 câu hỏi thành công trên [grading_run.jsonl](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/eval/grading_run.jsonl).

---

## 4. Freshness & monitoring (100–150 từ)

Nhóm cấu hình SLA là 24 giờ. Cảnh báo `freshness_check` được tính dựa trên thời gian xuất bản dữ liệu thô `latest_exported_at` so với thời gian thực của hệ thống.
*   **PASS:** Dữ liệu có thời gian cập nhật dưới 24 giờ, hệ thống hoạt động bình thường, dữ liệu tươi mới.
*   **WARN:** Dữ liệu trễ từ 12-24 giờ. Cần theo dõi nguồn cấp dữ liệu.
*   **FAIL:** Trễ trên 24 giờ. Nguồn cấp dữ liệu DB/API bị gián đoạn, cần restart hệ thống đồng bộ dữ liệu. Manifest mẫu của dữ liệu lịch sử (tháng 4/2026) trả về FAIL do độ trễ đã đạt hơn 1400 giờ, đây là phản hồi chính xác của SLA checker.

---

## 5. Liên hệ Day 09 (50–100 từ)

Dữ liệu được nạp vào collection `day10_kb` hoàn toàn có thể được tích hợp trực tiếp cho các RAG agent ở Day 09. Thay vì để agent đọc trực tiếp các tài liệu thô tĩnh dạng text, việc hướng agent truy vấn trực tiếp vào collection ChromaDB đã qua pipeline dọn dẹp này giúp loại bỏ hoàn toàn các câu trả lời sai lệch do tài liệu stale hoặc dữ liệu bị trùng lặp gây nhiễu ngữ cảnh.

---

## 6. Rủi ro còn lại & việc chưa làm

*   **Dịch vụ tải model:** SentenceTransformers tải model online qua Hugging Face có thể bị nghẽn mạng. Cần cấu hình lưu cache model local.
*   **Cảnh báo Slack:** Hiện tại việc cảnh báo mới chỉ ghi log tĩnh, chưa cấu hình Slack Webhook gửi alert tự động theo thời gian thực khi freshness check FAIL hoặc pipeline halt.
*   **LLM-judge:** Chưa xây dựng LLM-judge tự động để đánh giá ngữ nghĩa câu trả lời của agent sau khi retrieval.
