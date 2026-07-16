# CDO07 Research — Phân tích giải pháp phân tích log

> **Vị trí:** `C:\Users\NguyenPhuTrieu\Downloads\research-cdo07-log-analysis`  
> Thư mục **ngoài** git repo `tf4-phase3-repo` — không ảnh hưởng PR/evidence Mandate-04.  
> Evidence gốc vẫn nằm trong repo: `docs/evidence/mandate-04-forensic/`.

| | |
|---|---|
| **Nhóm** | CDO07 |
| **Ngày** | 16/07/2026 |
| **Task** | Phân tích các giải pháp phân tích log hiện tại → đưa backlog |
| **Giải pháp đề xuất** | **Athena (P0)** + CloudWatch Insights/Grafana (P1) + OpenSearch audit index (P2 tùy chọn) |

---

## Đọc theo thứ tự

| # | File | Mục đích |
|---|---|---|
| 0 | [00-ban-do-mandate-04-forensic.md](./00-ban-do-mandate-04-forensic.md) | Bản đồ Mandate-04 Forensic |
| 1 | [01-hien-trang-mandate-04-forensic.md](./01-hien-trang-mandate-04-forensic.md) | Hiện trạng + điểm đau |
| 2 | [02-so-sanh-giai-phap-phan-tich-log.md](./02-so-sanh-giai-phap-phan-tich-log.md) | So sánh OpenSearch / ELK / Loki / Datadog / Athena |
| 3 | [03-de-xuat-backlog.md](./03-de-xuat-backlog.md) | Stories backlog + DoD |
| 4 | [04-giai-phap-de-xuat.md](./04-giai-phap-de-xuat.md) | **Giải pháp kiến trúc + PoC + query mẫu** |
| 5 | [05-ADR-chon-lop-phan-tich-log.md](./05-ADR-chon-lop-phan-tich-log.md) | ADR chờ nhóm chốt |
| — | [06-script-trinh-bay-5-phut.md](./06-script-trinh-bay-5-phut.md) | Script nói chuyện họp |

---

## Một câu tóm tắt

Mandate-04 đã **xong ghi + khóa log** (S3 Object Lock).  
Task này thêm lớp **đọc/phân tích**: ưu tiên **Athena query thẳng S3** (hết cảnh download), Grafana/Insights cho sự kiện gần, OpenSearch chỉ sau khi bảo mật ổn — **không** dựng ELK/Datadog làm SoT.
