# Báo cáo cá nhân — mẫu GV (reference)

**Họ và tên:** GV Reference  
**Vai trò:** Cleaning & Quality  
**Độ dài:** ~450 từ (mẫu)

---

## 1. Phần phụ trách cụ thể

Trong bài lab này, tôi phụ trách hai phần chính: chuẩn hoá dữ liệu đầu vào ở `transform/cleaning_rules.py` và kiểm soát chất lượng ở `quality/expectations.py`. Mục tiêu là đảm bảo dữ liệu trước khi embed vào vector store không còn các lỗi dễ gây “hallucination theo ngữ cảnh”, đặc biệt là xung đột policy refund (7 ngày vs 14 ngày) và policy nghỉ phép HR (12 ngày vs 10 ngày).

Cụ thể, tôi bổ sung các rule xử lý dữ liệu bẩn như loại BOM ở đầu chuỗi, chuẩn hoá khoảng trắng nội bộ, và trim `doc_id` để tránh rơi vào `unknown_doc_id` giả. Tôi cũng phối hợp với phần pipeline để đảm bảo record bị loại luôn có `reason` trong quarantine CSV, giúp truy vết và audit nhanh khi có incident.

---

## 2. Một quyết định kỹ thuật quan trọng

Quyết định quan trọng nhất là đặt một số expectation ở mức `halt` thay vì `warn`. Ví dụ:

- `refund_no_stale_14d_window` (không được còn cụm 14 ngày trong policy hiện hành)
- `exported_at_present_all_cleaned_rows` (mọi dòng cleaned phải có watermark thời gian)

Lý do: nếu chỉ cảnh báo, pipeline vẫn publish vào index và downstream retrieval có thể trả lời “nhìn đúng top-1” nhưng vẫn kéo theo context cũ trong top-k. Với dữ liệu policy, sai ngữ cảnh là lỗi nghiêm trọng hơn lỗi style, nên `halt` giúp chặn publish boundary rõ ràng hơn.

---

## 3. Một sự cố/anomaly và cách xử lý

Sự cố điển hình là sau khi chạy inject có chủ đích (`--no-refund-fix --skip-validate`), kết quả grading cho thấy `gq_d10_01` có `contains_expected=true` nhưng `hits_forbidden=true`. Điều này nghĩa là hệ thống vẫn tìm được câu đúng “7 ngày”, nhưng trong top-k còn tồn tại chunk “14 ngày làm việc”.

Quy trình xử lý:

1. Chạy lại pipeline chuẩn không skip validate.
2. Kiểm tra log có `embed_prune_removed` để xác nhận vector stale đã bị dọn.
3. Chạy lại `grading_run.py` và `instructor_quick_check.py`.

Kết quả sau fix: cả ba câu `gq_d10_01..03` đều đạt `MERIT_CHECK OK`.

---

## 4. Bằng chứng before/after

- **Before (inject):** `artifacts/eval/after_inject_bad.csv` cho `q_refund_window` có `hits_forbidden=yes`.
- **After (clean run):** `artifacts/eval/before_after_eval.csv` cho cùng câu có `hits_forbidden=no`.

Log run tốt có thể tham chiếu:

- `artifacts/logs/run_fix-pass-debug.log` (hoặc run chuẩn tương đương)
- `artifacts/manifests/manifest_fix-pass-debug.json`

Hai artifact này đủ để chứng minh dữ liệu đã qua clean + validate + publish nhất quán.

---

## 5. Hướng cải tiến nếu có thêm 2 giờ

Nếu có thêm thời gian, tôi sẽ mở rộng quan sát chất lượng theo hai hướng:

1. Đo freshness ở cả 2 boundary (ingest và publish) thay vì chỉ dùng một timestamp tổng.
2. Viết test tự động cho một số rule nhạy cảm (BOM, stale keyword, version conflict) để tránh regression khi thay đổi cleaning logic.

Mục tiêu là chuyển từ “pass lab” sang “vận hành ổn định” khi dữ liệu nguồn thay đổi liên tục.
