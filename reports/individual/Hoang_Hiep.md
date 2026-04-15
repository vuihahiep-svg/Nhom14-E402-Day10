# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Hoàng Hiệp
**Vai trò:** Cleaning & Quality Owner 
**Ngày nộp:** 2026-04-15  
**Độ dài:** ~520 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `transform/cleaning_rules.py` — thêm ba rule có tên rõ: `trim_doc_id_whitespace`, `strip_utf8_bom_from_chunk_text`, `collapse_internal_whitespace_in_chunk_text` (chuỗi xử lý trước allowlist và dedupe).  
- `quality/expectations.py` — thêm hai expectation halt: `exported_at_present_all_cleaned_rows`, `cleaned_at_least_three_distinct_doc_ids`.

**Kết nối với thành viên khác:** rule clean quyết định nội dung đưa vào `cmd_embed_internal` trong `etl_pipeline.py`; expectation quyết định có được publish hay dừng ở ranh giới validate.

**Bằng chứng:** commit trong repo lab; log `run_id=sprint2-clean` có đủ dòng expectation mới OK.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi chọn **cả hai expectation mới đều `severity=halt`** thay vì cảnh báo, vì chúng bảo vệ **watermark vận hành** và **đa dạng tài liệu** — nếu chỉ warn, pipeline vẫn embed và agent có thể trả lời trên index “một doc” hoặc không có `exported_at` để đối chiếu freshness. Đánh đổi: inject Sprint 3 **không thể** vừa bỏ refund fix vừa giữ halt (đúng — cần `--skip-validate` có kiểm soát). Ranh giới publish rõ: halt = không embed trừ khi operator ghi nhận rủi ro.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** sau khi chạy `inject-bad`, `eval_retrieval` cho `q_refund_window` báo `hits_forbidden=yes` dù `contains_expected=yes`.  
**Phát hiện:** README và `eval_retrieval.py` quét **toàn bộ top-k** cho `must_not_contain`, nên chunk stale “14 ngày làm việc” vẫn làm fail quan sát được.  
**Fix:** chạy lại pipeline chuẩn (`python etl_pipeline.py run --run-id sprint2-clean`) để áp rule refund 14→7 và để expectation pass; Chroma prune+upsert loại vector không còn trong cleaned.  
**Metric:** `hits_forbidden` chuyển từ `yes` → `no` trên `artifacts/eval/before_after_eval.csv`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**run_id:** `inject-bad` (trước) vs `sprint2-clean` (sau).

- Trước — dòng `q_refund_window` trong `artifacts/eval/after_inject_bad.csv`: `contains_expected=yes`, `hits_forbidden=yes`.  
- Sau — cùng câu trong `artifacts/eval/before_after_eval.csv`: `contains_expected=yes`, `hits_forbidden=no`.

Kèm log: `expectation[refund_no_stale_14d_window] FAIL` trong `artifacts/logs/run_inject-bad.log` đối lập `OK` trong `run_sprint2-clean.log`.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu thêm ~2 giờ, tôi sẽ đọc `exported_at` từ contract/env để scenario **FAIL freshness có chủ đích** trên cùng CSV (demo SLA “snapshot”) mà không đổi clock hệ thống — phục vụ phần bonus đo hai boundary trong `SCORING.md`. Tôi cũng sẽ thêm một test nhỏ (pytest) cố định hành vi dedupe sau BOM để regression không lặp lại âm thầm.
