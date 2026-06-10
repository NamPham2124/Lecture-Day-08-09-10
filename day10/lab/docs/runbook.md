# Runbook — Lab Day 10

## Symptom

User/agent trả lời policy cũ, ví dụ “14 ngày làm việc” cho refund thay vì 7 ngày, hoặc “10 ngày phép năm” cho HR 2026 thay vì 12 ngày. Grading có thể báo `hits_forbidden=true` hoặc `top1_doc_matches=false`.

## Detection

- Kiểm tra log `artifacts/logs/run_<run_id>.log` cho expectation `FAIL`.
- Chạy `python grading_run.py --out artifacts/eval/grading_run.jsonl` và kiểm `contains_expected`, `hits_forbidden`, `top1_doc_matches`.
- Chạy freshness trên manifest: `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_codex-final.json`.

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Mở `artifacts/manifests/manifest_<run_id>.json` | Thấy `raw_records`, `cleaned_records`, `quarantine_records`, `latest_exported_at` |
| 2 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | Có reason rõ cho record bị loại |
| 3 | Chạy `python eval_retrieval.py --out artifacts/eval/eval_after_fix.csv` | Tất cả `contains_expected=yes`, `hits_forbidden=no` |
| 4 | So sánh inject bad | `codex-inject-bad` có refund expectation fail và grading `gq_d10_01 hits_forbidden=true` |

## Mitigation

Rerun pipeline chuẩn không dùng `--skip-validate`: `python etl_pipeline.py run --run-id codex-final`. Nếu đang dùng Chroma, publish sẽ upsert theo `chunk_id` và prune id cũ. Nếu thiếu dependency, fallback index JSON được ghi đè bằng snapshot sạch.

## Prevention

- Giữ `access_control_sop` trong allowlist và contract.
- Duy trì expectation halt cho refund 14 ngày, HR 10 ngày, coverage đủ 5 nguồn grading.
- Khi thêm source/version mới, cập nhật contract, allowlist và test questions cùng lúc.
- Không dùng `--skip-validate` ngoài kịch bản inject evidence.
