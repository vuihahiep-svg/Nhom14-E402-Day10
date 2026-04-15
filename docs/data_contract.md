# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| **Export CSV** (`data/raw/policy_export_dirty.csv`) | `etl_pipeline.py` → `load_raw_csv` | Trùng `chunk_text`, BOM/whitespace lạ, `doc_id` lệch catalog, ngày không ISO, bản HR/refund cũ xen kẽ | `raw_records`, `quarantine_records`, log `reason` trong `artifacts/quarantine/*.csv` |
| **Canonical text** (`data/docs/*.txt`) | Con người / CMS (ngoài scope pipeline lab); đồng bộ `doc_id` với allowlist | Sai version policy so với export; file không khớp `canonical_sources` trong YAML | Soát diff nội dung định kỳ; expectation `refund_no_stale_14d_window`, `hr_leave_no_stale_10d_annual` |
| **Chroma publish** (`CHROMA_DB_PATH`, collection `day10_kb`) | Embed sau validate | Vector lạc hậu sau publish; top-k còn chunk cấm | `embed_prune_removed`, `eval_retrieval.py` (`hits_forbidden`), `grading_run.jsonl` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | SHA256 rút gọn + `doc_id` + seq — idempotent upsert |
| doc_id | string | Có | Thuộc `allowed_doc_ids` trong contract |
| chunk_text | string | Có | Sau BOM strip, whitespace collapse, refund window fix (tuỳ flag) |
| effective_date | date (YYYY-MM-DD) | Có | Chuẩn hoá từ ISO hoặc DD/MM/YYYY |
| exported_at | datetime | Có | Dùng manifest `latest_exported_at` + expectation `exported_at_present_all_cleaned_rows` |

---

## 3. Quy tắc quarantine vs drop

Record không đạt allowlist / parse ngày / trùng nội dung / HR cũ / text rỗng được **ghi vào `artifacts/quarantine/<run>.csv`** với cột `reason`. Không xoá silently khỏi nguồn — operator xem quarantine, sửa export hoặc cập nhật catalog rồi rerun pipeline.

---

## 4. Phiên bản & canonical

**Refund:** `policy_refund_v4` — cửa sổ hoàn tiền **7 ngày làm việc** sau clean (expectation halt nếu còn “14 ngày làm việc” khi đã bật fix).  
**HR leave:** bản có `effective_date >= 2026-01-01` và không còn wording “10 ngày phép năm” trên doc HR trong cleaned (bản 2025 vào quarantine theo ngày hiệu lực).
