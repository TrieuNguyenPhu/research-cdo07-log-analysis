# CDO07 Research — Phân tích giải pháp phân tích log

> **Chỉnh sửa tại:** `C:\Users\NguyenPhuTrieu\Downloads\research-cdo07-log-analysis`  
> **Nguồn kỹ thuật:** luôn quét `D:\source code\tf4-phase3-repo`

## File trình bày chính (chốt)

**→ [PHUONG-AN-CHOT-phan-tich-log-CDO07.md](./PHUONG-AN-CHOT-phan-tich-log-CDO07.md)**  
Phương án tốt nhất đã chốt: **Athena + CloudWatch Logs Insights trên SoT S3 Object Lock**.  
Dùng file này để trình bày / đưa backlog.

| | |
|---|---|
| **Quyết định** | Athena (đọc S3) + Insights (drill gần); không ELK/Datadog; OpenSearch audit = P2 |
| **Cost** | Ceiling ≤ $5/tuần; kỳ vọng <$1/tuần |
| **Ngày** | 16/07/2026 |

---

## Tài liệu nền (chi tiết / lịch sử nghiên cứu)

| # | File | Mục đích |
|---|---|---|
| 0 | [00-ban-do…](./00-ban-do-mandate-04-forensic.md) | Bản đồ Mandate-04 |
| 1 | [01-hien-trang…](./01-hien-trang-mandate-04-forensic.md) | Hiện trạng + pain |
| 2 | [02-so-sanh…](./02-so-sanh-giai-phap-phan-tich-log.md) | So sánh giải pháp |
| 3 | [03-de-xuat-backlog…](./03-de-xuat-backlog.md) | Stories backlog |
| 4–8 | `04`…`08` | Giải pháp nháp, ADR, script, sync, cost |
| 7 | [07-sync…](./07-sync-repo-2026-07-16.md) | Nhật ký quét repo |
