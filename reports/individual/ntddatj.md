# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Tiến Đạt  
**Mã số học viên (MSHV):** 2A202600595  
**Vai trò:** Ingestion / Cleaning Owner  
**Ngày nộp:** 2026-06-10  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong dự án Lab Day 10, tôi đảm nhiệm vai trò **Ingestion / Cleaning Owner**. 
*   **File / module phụ trách chính:** Tôi trực tiếp chỉnh sửa và mở rộng logic làm sạch dữ liệu trong [cleaning_rules.py](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/transform/cleaning_rules.py) và khai báo cấu trúc data contract tại [data_contract.yaml](file:///c:/Users/ntddatj/github-ntddatj/github-classroom/Lecture-Day-08-09-10/day10/lab/contracts/data_contract.yaml).
*   **Kết nối với thành viên khác:** Tôi làm việc chặt chẽ với Quality Owner để cung cấp dữ liệu sạch đầu ra khớp với các expectations kiểm tra chất lượng, đồng thời phối hợp với Embed Owner để đảm bảo định dạng file CSV sạch phục vụ nạp vector store chính xác.
*   **Bằng chứng:** Tôi đã thêm tài liệu mới `access_control_sop` vào danh sách cho phép, phát triển 5 quy tắc dọn dẹp mở rộng và sửa lại luồng loại trùng lặp (deduplication) sau chuẩn hóa.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất tôi đưa ra là **đảo luồng kiểm tra trùng lặp (deduplication check) xuống sau khi dọn dẹp văn bản** thay vì để ở đầu hàm như bản baseline.
*   *Lý do:* Trong dữ liệu thô DB/API xuất ra, có rất nhiều tài liệu có nội dung ngữ nghĩa giống hệt nhau nhưng bị chèn thêm các ký tự rác (như dấu `!!!`, các ghi chú hệ thống khác nhau, hoặc các tiền tố boilerplate). Nếu thực hiện deduplicate dựa trên chuỗi text thô ban đầu, các bản ghi này sẽ có mã hash khác nhau và đều vượt qua bộ lọc để đi vào cơ sở dữ liệu vector.
*   *Giải pháp:* Tôi thực hiện chuẩn hóa văn bản trước (loại bỏ dấu than, tiền tố boilerplate, chuẩn hóa dấu cách), sau đó mới thực hiện so khớp trùng lặp. Việc này giúp loại bỏ triệt để các chunk trùng lặp ngữ nghĩa, giảm nhiễu dữ liệu và tiết kiệm không gian lưu trữ cho ChromaDB.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

*   **Triệu chứng:** Khi chạy chấm điểm kiểm thử hệ thống RAG, câu số 6 (`gq_d10_06`) về cơ chế tự động chuyển tiếp sự cố (auto escalate) liên tục trả về kết quả sai (`contains_expected: false`).
*   **Phát hiện:** Tôi đã viết đoạn script truy vấn trực tiếp ChromaDB để kiểm tra top-10 kết quả gần nhất của câu hỏi này. Kết quả chỉ ra rằng chunk chứa từ khóa đúng ("10 phút") bị xếp hạng thứ 8. Nguyên nhân do database bị lấn át bởi nhiều bản ghi trùng lặp của chính sách SLA P1 thô chưa được gộp. Đồng thời, câu hỏi dùng thuật ngữ tiếng Anh ("auto escalate") trong khi tài liệu dùng tiếng Việt ("tự động escalate").
*   **Khắc phục:** Tôi đã sửa thứ tự dọn dẹp dữ liệu để loại trùng hiệu quả và thêm quy tắc chuẩn hóa thuật ngữ (Terminology Standardization) tự động bổ sung cụm từ tiếng Anh tương đương `(auto escalate)` vào tài liệu. Sau khi chạy lại pipeline, tài liệu đúng đã vươn lên hạng 3 và vượt qua bài kiểm tra chấm điểm.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Bằng chứng thực tế trích xuất từ file kết quả đánh giá với `run_id` `2026-06-10T10-14Z`:

*   **Trước khi sửa (Trạng thái dữ liệu lỗi - inject-bad):**
    `q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền...,policy_refund_v4,"Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc...",yes,yes,yes,3`
    *(Nhận xét: `hits_forbidden` báo `yes` do chứa thông tin stale "14 ngày")*

*   **Sau khi sửa (Trạng thái sạch chuẩn):**
    `q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền...,policy_refund_v4,"Yêu cầu hoàn tiền được chấp nhận trong vòng 7 ngày làm việc... [cleaned: stale_refund_window]",yes,no,yes,3`
    *(Nhận xét: `hits_forbidden` báo `no`, thông tin đã được chuẩn hóa về "7 ngày")*

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ phát triển một giao diện web Dashboard đơn giản (sử dụng Gradio/Streamlit) để hiển thị trực quan tỷ lệ phân bổ các lý do dữ liệu bị cách ly (quarantine reasons) theo thời gian thực. Điều này giúp đội ngũ vận hành nhanh chóng nhận biết được hệ thống nguồn nào đang gặp lỗi xuất bản dữ liệu nhiều nhất.
