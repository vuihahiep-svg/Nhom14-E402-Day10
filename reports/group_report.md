# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Lab Day 10 (demo repo)  
**Thành viên:**

| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Hoàng Hiệp | Cleaning & Quality Owner  | hiep.cbla5@gmail.com |

**Ngày nộp:** 2026-04-15  
**Repo:** `Lecture-Day-08-09-10/day10/lab`

---

## 1. Pipeline tổng quan (150–200 từ)

Nguồn raw là export CSV mẫu `data/raw/policy_export_dirty.csv` (13 dòng) mô phỏng lỗi thực tế: trùng lặp, ngày không ISO, `doc_id` lạ, HR cũ (10 ngày phép), refund stale (14 ngày), BOM/whitespace và `doc_id` có khoảng trắng thừa. Pipeline `etl_pipeline.py` đọc raw, chạy `clean_rows`, ghi `artifacts/cleaned/` và `artifacts/quarantine/`, chạy `run_expectations`, embed Chroma (`day10_kb`) với upsert + prune theo `chunk_id`, rồi ghi manifest JSON và log freshness.

**Lệnh một dòng (Definition of Done README):**

```bash
cd day10/lab && python etl_pipeline.py run
```

**Sprint 1 — log có đủ metric:** ví dụ `run_id=sprint1` trong `artifacts/logs/run_sprint1.log` với `raw_records=13`, `cleaned_records=7`, `quarantine_records=6`.

---

## 2. Cleaning & expectation (150–200 từ)

Baseline giữ nguyên: allowlist `doc_id`, chuẩn hoá `effective_date`, quarantine HR cũ theo cutoff 2026, dedupe theo nội dung, fix refund 14→7 khi bật flag mặc định. Nhóm thêm **3 rule** trong `transform/cleaning_rules.py` (mỗi rule có tên trong docstring) và **2 expectation halt mới** trong `quality/expectations.py`.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `trim_doc_id_whitespace` | Nếu không strip: 1 dòng `policy_refund_v4 ` → `unknown_doc_id` | Với strip: dòng 13 vào **cleaned** (thay vì quarantine) | `artifacts/quarantine/quarantine_sprint2-clean.csv` không còn `unknown_doc_id` cho doc đó; so sánh với baseline tắt rule (tư duy lab) |
| `strip_utf8_bom_from_chunk_text` + `collapse_internal_whitespace` | BOM / khoảng trắng kép tạo “gần trùng” | Thêm **duplicate_chunk_text** cho dòng 11–12 → `quarantine_records` tăng trên bộ 13 dòng | `quarantine_sprint2-clean.csv` có `reason=duplicate_chunk_text` cho chunk BOM / SLA spacing |
| `exported_at_present_all_cleaned_rows` (halt) | Nếu xoá `exported_at` trên 1 dòng cleaned → **halt** | Trên CSV mẫu: OK | Log expectation `empty_exported_at_count=0` |
| `cleaned_at_least_three_distinct_doc_ids` (halt) | Nếu ingest lỗi chỉ còn 1–2 doc → **halt** | Trên CSV mẫu: `distinct_doc_ids=4` | Log dòng `expectation[cleaned_at_least_three_distinct_doc_ids]` |

**Ví dụ expectation fail (demo có chủ đích):** `inject-bad` → `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1`; tiếp tục chỉ khi `--skip-validate` (ghi trong `docs/runbook.md`).

---

## 3. Before / after ảnh hưởng retrieval (200–250 từ)

**Kịch bản inject:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` embed bản cleaned **còn** cụm “14 ngày làm việc”.  
**Eval:** `python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv`.

**Kết quả định lượng:** với `q_refund_window`, file `after_inject_bad.csv` ghi `contains_expected=yes` nhưng `hits_forbidden=yes` — đúng tinh thần observability “top-1 nhìn ổn nhưng context vẫn độc”. Sau `sprint2-clean` + eval `before_after_eval.csv`: `hits_forbidden=no`.  
Đối với **Merit** `q_leave_version`: trước/sau đều `yes/no/yes` trên các cột tương ứng vì HR cũ đã bị quarantine theo ngày hiệu lực; chứng cứ nằm ở quarantine + expectation `hr_leave_no_stale_10d_annual`.

---

## 4. Freshness & monitoring (100–150 từ)

`monitoring/freshness_check.py` đọc `latest_exported_at` trên manifest so với SLA giờ (`FRESHNESS_SLA_HOURS`, mặc định 24). Trên dữ liệu lab đã chỉnh `exported_at` về `2026-04-15T08:00:00`, check thường **PASS** khi chạy gần ngày lab. Nếu FAIL: hiểu là snapshot export quá cũ so với SLA đã chọn — ghi rõ trong runbook (snapshot vs pipeline run).

---

## 5. Liên hệ Day 09 (50–100 từ)

Pipeline Day 10 cung cấp **lớp dữ liệu đã publish** (Chroma collection `day10_kb`) mà agent Day 09 có thể trỏ retrieval vào cùng narrative CS/IT. Ranh giới publish là manifest + eval trước khi bật tool search trong agent.

---

## 6. Rủi ro còn lại & việc chưa làm

- Chưa tích hợp Great Expectations đầy đủ (chỉ suite custom).  
- Chưa tách đo freshness ở hai boundary (ingest vs publish) cho bonus Distinction.
