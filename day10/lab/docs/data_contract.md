# Data contract — Lab Day 10

File YAML đồng bộ: `contracts/data_contract.yaml`.

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | CSV export từ policy/refund-v4 | Stale refund window 14 ngày, duplicate chunk | `refund_no_stale_14d_window`, `hits_forbidden` |
| `sla_p1_2026` | CSV export từ support/sla-p1-2026 | Missing effective date, duplicate SLA chunk | `effective_date_iso_yyyy_mm_dd`, eval P1 |
| `it_helpdesk_faq` | CSV export từ support/helpdesk-faq | Empty chunk, stale/malformed date | `chunk_min_length_8`, eval FAQ |
| `hr_leave_policy` | CSV export từ hr/leave-policy-2026 | Conflict HR 2025 10 ngày vs HR 2026 12 ngày | `hr_leave_no_stale_10d_annual`, `hr_has_current_12d_annual` |
| `access_control_sop` | CSV export từ it/access-control-sop | Missing allowlist source | `grading_sources_coverage`, grading gq_d10_10 |

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| `chunk_id` | string | Có | Stable hash từ `doc_id`, `chunk_text`, sequence |
| `doc_id` | string | Có | Phải thuộc allowlist trong contract/code |
| `chunk_text` | string | Có | Đã strip prefix lỗi, collapse token lặp, loại empty |
| `effective_date` | date | Có | Chuẩn ISO `YYYY-MM-DD` |
| `exported_at` | datetime | Có | Dùng để tính freshness |

## 3. Quy tắc quarantine vs drop

Record không đạt rule được ghi vào `artifacts/quarantine/quarantine_<run_id>.csv` với `reason`, không drop âm thầm. Các lý do chính: `unknown_doc_id`, `missing_effective_date`, `invalid_effective_date_format`, `stale_hr_content_marker`, `missing_chunk_text`, `duplicate_chunk_text`. Owner nguồn dữ liệu phải approve trước khi đưa record quarantine quay lại cleaned.

## 4. Phiên bản & canonical

Canonical refund là `data/docs/policy_refund_v4.txt`, cửa sổ hiện hành là 7 ngày làm việc. Canonical HR là `data/docs/hr_leave_policy.txt`, cutoff `hr_leave_min_effective_date=2026-01-01`, nội dung hiện hành cho nhân viên dưới 3 năm là 12 ngày phép năm. `access_control_sop` là nguồn hợp lệ và đã được thêm vào allowlist.
