# Quality report — Lab Day 10 (nhóm)

**run_id (publish chuẩn):** `sprint2-clean`  
**run_id (inject Sprint 3):** `inject-bad`  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (inject-bad, embed xấu) | Sau (sprint2-clean) | Ghi chú |
|--------|------------------------------|---------------------|---------|
| raw_records | 13 | 13 | Cùng export `data/raw/policy_export_dirty.csv` |
| cleaned_records | 7 | 7 | Cùng rule clean; khác ở chỗ có/không fix refund |
| quarantine_records | 6 | 6 | BOM + whitespace + trim `doc_id` tạo thêm dòng vào quarantine so với baseline thuần |
| Expectation halt? | Có (`refund_no_stale_14d_window` FAIL) — bỏ qua bằng `--skip-validate` | Không — suite full OK | Log: `artifacts/logs/run_inject-bad.log`, `run_sprint2-clean.log` |

---

## 2. Before / after retrieval (bắt buộc)

**File eval inject (trước khi fix index):** `artifacts/eval/after_inject_bad.csv`  
**File eval sau pipeline chuẩn:** `artifacts/eval/before_after_eval.csv`

**Câu then chốt — `q_refund_window`**

- **Trước (inject, top-k=3):** `contains_expected=yes`, `hits_forbidden=yes` — top-1 đúng 7 ngày nhưng **toàn bộ top-k vẫn chứa** cụm cấm `14 ngày làm việc` từ chunk stale còn trong index.  
- **Sau (sprint2-clean):** `contains_expected=yes`, `hits_forbidden=no`.

**Merit — `q_leave_version`**

- **Trước / Sau:** `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` — HR cũ (10 ngày) đã vào quarantine theo `effective_date`; index chỉ giữ bản 2026 (12 ngày).

---

## 3. Freshness & monitor

Manifest `artifacts/manifests/manifest_sprint2-clean.json`: `freshness_check=PASS` với `latest_exported_at=2026-04-15T08:00:00` và SLA mặc định 24h (`FRESHNESS_SLA_HOURS`). Nếu snapshot cũ hơn SLA, trạng thái có thể **FAIL** — đó là tín hiệu “export nguồn không được refresh”, không nhất thiết lỗi code (xem `docs/runbook.md`).

---

## 4. Corruption inject (Sprint 3)

Chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` để **giữ text 14 ngày** trong cleaned và **bỏ qua halt** rồi vẫn embed — mô phỏng migration sai + vận hành nguy hiểm. Kỳ vọng: `eval_retrieval` báo `hits_forbidden=yes` cho `q_refund_window`. Sau đó chạy lại pipeline chuẩn (fix refund + validate) để khôi phục index.

---

## 5. Hạn chế & việc chưa làm

- Eval keyword không thay LLM-judge; có thể mở rộng sau.  
- Freshness đơn giản theo một timestamp trên manifest — chưa đo tách **ingest boundary** vs **publish boundary** (gợi ý bonus trong `SCORING.md`).
