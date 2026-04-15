# Runbook — Lab Day 10 (incident tối giản)

**Một lệnh chạy pipeline chuẩn (ingest → clean → validate → embed → manifest):**

```bash
cd lab && python etl_pipeline.py run
```

**Kiểm tra freshness theo manifest vừa tạo:**

```bash
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run-id>.json
```

**PASS / WARN / FAIL (`monitoring/freshness_check.py`):**

- **PASS:** `latest_exported_at` (hoặc fallback `run_timestamp`) trong SLA `FRESHNESS_SLA_HOURS` (mặc định 24) so với “now” UTC.
- **WARN:** không parse được timestamp trong manifest (thiếu hoặc sai định dạng) — cần sửa manifest writer hoặc export.
- **FAIL:** tuổi dữ liệu (`age_hours`) vượt SLA — snapshot export quá cũ so với ngưỡng đã chọn; có thể hợp lý với dữ liệu lab cố định → ghi rõ SLA áp cho **snapshot** hay cho **lần chạy pipeline**.

---

## Symptom

Người dùng hoặc agent trả lời **“14 ngày làm việc”** cho hoàn tiền, hoặc **“10 ngày phép năm”** cho HR 2026, dù policy đã cập nhật.

---

## Detection

- **`expectation[refund_no_stale_14d_window] FAIL`** trong `artifacts/logs/run_*.log`.
- **`eval_retrieval.py`:** `q_refund_window` có `contains_expected=no` hoặc `hits_forbidden=yes`.
- **`freshness_check=FAIL`** trên manifest nếu export timestamp quá cũ (cảnh báo dữ liệu nguồn không được refresh).

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Mở `artifacts/manifests/manifest_<run-id>.json` | Có `run_id`, `cleaned_csv`, `quarantine_records`, `no_refund_fix`, `skipped_validate` |
| 2 | Mở `artifacts/quarantine/*.csv` | Thấy `reason` (duplicate, unknown_doc_id, stale_hr, …) giải thích mất chunk |
| 3 | Chạy `python eval_retrieval.py --out artifacts/eval/before_after_eval.csv` | CSV cho `q_refund_window`, `q_leave_version` khớp kỳ vọng keyword + `top1_doc_expected` |

---

## Mitigation

1. Rerun **`python etl_pipeline.py run`** (bật fix refund mặc định, không `--skip-validate`).  
2. Nếu cần chứng minh before: `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` rồi eval; sau đó chạy lại pipeline chuẩn và eval để có after.  
3. Nếu collection lệch: xoá collection hoặc tin cậy **prune + upsert** đã có trong `etl_pipeline.py`.

---

## Prevention

- Giữ **≥2 expectation halt** mới kèm baseline: `exported_at_present_all_cleaned_rows`, `cleaned_at_least_three_distinct_doc_ids`.  
- Thêm gate CI: `python etl_pipeline.py run` + `python eval_retrieval.py` trên branch trước merge.  
- Owner cập nhật `contracts/data_contract.yaml` khi thêm `doc_id`.  
- Ghi nhận SLA freshness nội bộ (snapshot vs run time).

**Peer review (3 câu hỏi — Phần E slide):**

1. `run_id` nào được coi là “publish boundary” cho incident?  
2. Khi expectation halt, có được phép embed không và trong điều kiện nào?  
3. Metric nào chứng minh index không còn chunk cấm sau publish?
