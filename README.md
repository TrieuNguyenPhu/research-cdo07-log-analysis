# CDO07 — Bộ đề xuất phân tích log (gửi nhóm)

> Ngoài repo · 16/07/2026 · **Chưa merge** `tf4-phase3-repo`  
> Đã rà soát đồng bộ tên Glue DB / IAM owner / ticket (cùng ngày).

## Đọc gì khi họp

| Ưu tiên | File | Dùng khi |
|---|---|---|
| **1** | [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md) | Trình bày ngắn + xin duyệt (§1, §9, §10) |
| **2** | [05-architecture-athena-forensics.md](./05-architecture-athena-forensics.md) | Kiến trúc · DDL · query · cost |
| **3** | [02-phu-luc-so-sanh.md](./02-phu-luc-so-sanh.md) | Vì sao không ELK / Datadog / OpenSearch ngay |
| **4** | [tickets/AUDIT-015](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md) | Gửi CDO08 (IAM — blocker) |
| **5** | [tickets/AUDIT-014](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md) | PoC CDO07 sau khi có IAM |

## Quyết định một dòng

**Athena (`tf4_audit_forensics`) + Insights (CWL)** trên SoT Object Lock · không ELK/Datadog · OpenSearch audit = P2.

## Chuẩn đặt tên (toàn bộ package)

| Mục | Giá trị |
|---|---|
| Glue DB | `tf4_audit_forensics` |
| Workgroup | `tf4-cdo07-audit` |
| Result S3 | `s3://tf4-athena-query-results-511825856493/cdo07/` |
| IAM | AUDIT-015 → **CDO08** |
| PoC | AUDIT-014 → **CDO07** |
| Cost ceiling | ≤ $5/tuần (kỳ vọng <$1/tuần) |

## Cách họp 5 phút

1. `01` §1 + §9 + §10  
2. DDL/cost → `05`  
3. Duyệt → gửi **AUDIT-015** (CDO08) rồi làm **AUDIT-014** (CDO07)

## Checklist trước khi gửi nhóm

- [x] Phương án ngắn (`01`) + kiến trúc/DDL (`05`) + so sánh (`02`)
- [x] Ticket AUDIT-014 / AUDIT-015 đúng format repo
- [x] Backlog + Jira snippet
- [x] Tên Glue DB / workgroup / IAM owner thống nhất
- [x] Không còn file nháp / review thừa
- [ ] Nhóm duyệt §10 của `01`
- [ ] Copy tickets vào `docs/audit/tickets/` khi được duyệt

## Nguồn repo (chỉ đọc)

- Evidence: `docs/evidence/mandate-04-forensic/`
- Playbook: `docs/audit/runbooks/forensic-playbook-timeline.md`
- Config buckets: `infra/terraform/aws-config-storage.tf`
- Ticket mẫu: `docs/audit/tickets/AUDIT-010-*.md`, `AUDIT-013-*.md`
