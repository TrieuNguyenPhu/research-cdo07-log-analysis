# CDO07 — Bộ đề xuất phân tích log (gửi nhóm)

> Ngoài repo · 16/07/2026 · **Chưa merge** `tf4-phase3-repo`

## Đọc gì khi họp

| Ưu tiên | File | Dùng khi |
|---|---|---|
| **1** | [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md) | Trình bày ngắn + xin duyệt |
| **2** | [05-architecture-athena-forensics.md](./05-architecture-athena-forensics.md) | Bản kiến trúc / DDL / cost chi tiết |
| **3** | [02-phu-luc-so-sanh.md](./02-phu-luc-so-sanh.md) | Vì sao không ELK/Datadog/OpenSearch ngay |
| **4** | [tickets/AUDIT-014](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md) · [AUDIT-015](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md) | Ticket format repo |
| — | [03-backlog.md](./03-backlog.md) · [04-jira-tasks-snippet.md](./04-jira-tasks-snippet.md) | Backlog / dán JIRA_TASKS |

## Quyết định một dòng

**Athena (3 nguồn S3) + Insights (CWL)** trên SoT Object Lock · không ELK/Datadog · OpenSearch audit = P2.

## Cách họp 5 phút

1. `01` §1 + §9 + §10  
2. Ai hỏi DDL/cost chi tiết → `05`  
3. Duyệt → gửi AUDIT-015 (CDO08) rồi AUDIT-014 (CDO07)

## Nguồn repo (chỉ đọc)

- Evidence: `docs/evidence/mandate-04-forensic/`
- Playbook: `docs/audit/runbooks/forensic-playbook-timeline.md`
- Config buckets: `infra/terraform/aws-config-storage.tf`
