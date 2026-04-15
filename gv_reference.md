# Báo cáo cá nhân — mẫu GV (reference)

**Họ và tên:** Hoàng Hiệp  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài:** ~520 từ

---

## 1. Phụ trách

Tôi phụ trách chính `transform/cleaning_rules.py` và `quality/expectations.py`. Ở phần cleaning, tôi bổ sung các rule `trim_doc_id_whitespace`, `strip_utf8_bom_from_chunk_text`, `collapse_internal_whitespace_in_chunk_text` để ổn định dedupe và hạn chế lỗi catalog giả. Ở phần quality, tôi thêm hai expectation halt: `exported_at_present_all_cleaned_rows` và `cleaned_at_least_three_distinct_doc_ids`.

Kết nối với phần embed thông qua `cmd_embed_internal` trong `etl_pipeline.py` và các chỉ số theo run như `cleaned_records`, `quarantine_records`, `run_id` trên log/manifest.

**Bằng chứng:** log `run_id=sprint2-clean` có đầy đủ expectation mới ở trạng thái OK.

---

## 2. Quyết định kỹ thuật

Tôi chọn để cả hai expectation mới ở mức `halt` thay vì `warn`. Lý do là nếu cho phép publish khi thiếu watermark hoặc thiếu đa dạng doc_id, retrieval vẫn chạy nhưng rủi ro trả lời sai ngữ cảnh tăng cao. Halt giúp đặt ranh giới publish rõ ràng hơn, chỉ cho phép embed khi dữ liệu đạt tối thiểu tiêu chuẩn vận hành.

Về idempotency, tôi giữ cơ chế prune + upsert theo `chunk_id` trong Chroma để tránh giữ lại vector stale sau các lần inject.

---

## 3. Sự cố / anomaly

Khi chạy inject (`--run-id inject-bad --no-refund-fix --skip-validate`), kết quả retrieval cho `q_refund_window` xuất hiện tình huống `contains_expected=yes` nhưng `hits_forbidden=yes`. Nguyên nhân là top-k vẫn còn chunk chứa cụm cũ “14 ngày làm việc”.

Cách xử lý: chạy lại pipeline chuẩn `python etl_pipeline.py run --run-id sprint2-clean`, đảm bảo expectation pass và prune loại vector không còn trong cleaned.

---

## 4. Before/after

**Log:** `expectation[refund_no_stale_14d_window] FAIL` ở run `inject-bad`, sau đó chuyển thành `OK (halt)` ở run `sprint2-clean`.

**CSV:** ở `artifacts/eval/after_inject_bad.csv`, dòng `q_refund_window` có `hits_forbidden=yes`; sau fix, `artifacts/eval/before_after_eval.csv` ghi `hits_forbidden=no`.

---

## 5. Cải tiến thêm 2 giờ

Nếu có thêm thời gian, tôi muốn đọc cutoff HR từ `contracts/data_contract.yaml`/env thay vì hard-code trong Python, và thêm test regression cho các rule dedupe liên quan BOM/whitespace để giảm rủi ro phát sinh lại lỗi cũ.
