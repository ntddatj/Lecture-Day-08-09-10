# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom (Triệu chứng)

*   **RAG trả lời sai:** Người dùng hoặc chatbot agent nhận được thông tin lỗi thời (ví dụ: trả lời cửa sổ hoàn tiền là *"14 ngày"* thay vì *"7 ngày"*, hoặc thời gian nghỉ phép của nhân viên dưới 3 năm là *"10 ngày"* thay vì *"12 ngày"*).
*   **Thiếu thông tin:** Không tìm thấy câu trả lời cho các tài liệu mới (như quy trình Cấp quyền truy cập Admin - `access_control_sop`).

---

## Detection (Phát hiện)

*   **Freshness alert:** Manifest check báo `freshness_check=FAIL` (độ tươi dữ liệu vượt quá SLA cấu hình 24 giờ).
*   **Pipeline crash/halt:** Lệnh kiểm định chất lượng trả về `PIPELINE_HALT: expectation suite failed (halt)` khi chạy `etl_pipeline.py`.
*   **Evaluation regression:** Script `eval_retrieval.py` ghi nhận cột `hits_forbidden` giá trị `"yes"`.

---

## Diagnosis (Chẩn đoán)

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra file manifest gần nhất tại [artifacts/manifests/](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/manifests/) | Xác định `run_id`, cờ `no_refund_fix`, `skipped_validate` xem có bị kích hoạt nhầm hay không. Xem thời gian `latest_exported_at`. |
| 2 | Mở file quarantine tại [artifacts/quarantine/](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/artifacts/quarantine/) | Đọc cột `reason` để biết tại sao bản ghi bị cách ly (như `stale_hr_policy_text`, `unknown_doc_id`, `missing_effective_date`). |
| 3 | Chạy thử truy vấn đánh giá | Chạy `python eval_retrieval.py --out artifacts/eval/check_eval.csv` để xem các câu hỏi RAG bị fail và lý do không đạt. |

---

## Mitigation (Khắc phục)

1.  **Nếu do Ingest thiếu nguồn:** Bổ sung `doc_id` của nguồn tài liệu mới vào allowlist của [cleaning_rules.py](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/transform/cleaning_rules.py) và [data_contract.yaml](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/contracts/data_contract.yaml).
2.  **Nếu do dữ liệu stale (chứa thông tin cũ):** Sửa đổi dữ liệu tại nguồn hoặc bổ sung cleaning rules thay thế/loại bỏ. Chạy lại `python etl_pipeline.py run`.
3.  **Rollback dữ liệu:** Chạy lại pipeline với tệp dữ liệu sạch gần nhất để ghi đè lại vector store ChromaDB (nhờ cơ chế idempotent upsert và prune).

---

## Prevention (Phòng ngừa)

*   **Expectation Guardrails:** Duy trì bộ kiểm tra chất lượng dữ liệu tự động chặn nạp (`halt` severity) đối với các thông tin nhạy cảm (như phiên bản cũ của chính sách nghỉ phép, chính sách hoàn tiền).
*   **Cảnh báo Freshness:** Thiết lập cronjob chạy lệnh `python etl_pipeline.py freshness --manifest ...` định kỳ và gửi thông báo trực tiếp đến `#alerts-pipeline` trên Slack nếu độ tươi dữ liệu vượt SLA 24h.
