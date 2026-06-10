# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Day 10 Lab  
**Ngày nộp:** 2026-06-10  
**Run chính:** `codex-final`  
**Repo:** `Lecture-Day-08-09-10/day10/lab`

## 1. Pipeline tổng quan

Pipeline xử lý export raw `data/raw/policy_export_dirty.csv` gồm 247 record từ nhiều nguồn CS/IT/HR. Luồng chạy là ingest CSV, clean theo allowlist và rule versioning, validate bằng expectation suite, publish index, ghi manifest và freshness. Mỗi lần chạy ghi `run_id`, `raw_records`, `cleaned_records`, `quarantine_records`, đường dẫn cleaned/quarantine CSV và manifest. Run cuối `codex-final` tạo 38 cleaned records, 209 quarantine records, freshness PASS với SLA lab 2160 giờ.

Lệnh chạy end-to-end:

```bash
python etl_pipeline.py run --run-id codex-final
python eval_retrieval.py --out artifacts/eval/eval_after_fix.csv
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

## 2. Cleaning & expectation

Pipeline đã mở rộng allowlist với `access_control_sop`, chuẩn hóa ngày `DD/MM/YYYY` sang ISO, cách ly unknown doc_id, empty text/date, duplicate chunk, HR stale content và stale effective date. Refund window cũ “14 ngày làm việc” được sửa về “7 ngày làm việc” khi chạy chuẩn. Khi thiếu Chroma/SentenceTransformers, pipeline dùng fallback lexical index để vẫn tạo artifact trong môi trường lab nhẹ; nếu dependency có sẵn, Chroma vẫn là backend chính.

### 2a. Bảng metric_impact

| Rule / Expectation mới | Trước / inject | Sau clean | Chứng cứ |
|------------------------|----------------|-----------|----------|
| `access_control_sop` allowlist | `gq_d10_10` thiếu source nếu doc bị quarantine | `top1_doc_id=access_control_sop`, `top1_doc_matches=true` | `artifacts/eval/grading_run.jsonl` |
| `stale_hr_content_marker` | Raw có nhiều chunk HR 2025 “10 ngày phép năm” | `hr_leave_no_stale_10d_annual violations=0`; `gq_d10_09 hits_forbidden=false` | `artifacts/logs/run_codex-final.log` |
| `strip_unclear_prefix` | Raw có prefix `Nội dung không rõ ràng:` | `no_unclear_export_prefix unclear_prefix_rows=0` | `artifacts/logs/run_codex-final.log` |
| `grading_sources_coverage` | Thiếu source hợp lệ sẽ halt | `missing_doc_ids=[]` | `artifacts/logs/run_codex-final.log` |
| `refund_no_stale_14d_window` | Inject bad: `violations=2`, grading `gq_d10_01 hits_forbidden=true` | Clean: `violations=0`, grading `hits_forbidden=false` | `artifacts/logs/run_codex-inject-bad.log`, `artifacts/eval/grading_after_inject_bad.jsonl` |

Expectation halt mới/chính gồm `grading_sources_coverage`, `hr_has_current_12d_annual`, `refund_no_stale_14d_window`, `effective_date_iso_yyyy_mm_dd`. Expectation warn gồm `chunk_min_length_8` và `no_unclear_export_prefix`.

## 3. Before / after retrieval

Kịch bản inject chạy:

```bash
python etl_pipeline.py run --run-id codex-inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/eval_after_inject_bad.csv
python grading_run.py --out artifacts/eval/grading_after_inject_bad.jsonl
```

Ở inject bad, expectation `refund_no_stale_14d_window` fail với `violations=2` nhưng vẫn publish do `--skip-validate`. Grading evidence cho thấy `gq_d10_01` vẫn tìm được “7 ngày” nhưng `hits_forbidden=true` vì top-k context còn “14 ngày”. Sau khi rerun clean `codex-final`, `artifacts/eval/grading_run.jsonl` có đủ 10 dòng, tất cả `contains_expected=true`, `hits_forbidden=false`, và `top1_doc_matches=true`.

## 4. Freshness & monitoring

Manifest cuối: `artifacts/manifests/manifest_codex-final.json`. SLA đặt 2160 giờ trong lab vì snapshot mẫu có `exported_at` tháng 04/2026 trong khi ngày chạy là 2026-06-10. Freshness PASS nghĩa là snapshot còn nằm trong SLA học tập; production nên giảm SLA xuống 24 giờ hoặc đo thêm boundary ingest/publish.

## 5. Liên hệ Day 09

Day 09 agent/RAG phụ thuộc vào corpus đã publish. Pipeline này đảm bảo agent không đọc nhầm chunk stale như refund 14 ngày hoặc HR 10 ngày phép. Collection mặc định vẫn là `day10_kb`; fallback index chỉ dùng khi môi trường không cài Chroma.

## 6. Rủi ro còn lại & việc chưa làm

- Fallback lexical không thay thế embedding semantic trong production.
- Chưa tích hợp Great Expectations/pydantic model thật.
- Chưa có alert tự động ra Slack; hiện chỉ có log và artifact.
