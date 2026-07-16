# CDO07 — Bộ đề xuất phân tích log (gửi nhóm)

> Thư mục ngoài repo · Ngày 16/07/2026 · Task: phân tích giải pháp log → đưa backlog  
> **Repo `tf4-phase3-repo` không chứa các file này** — chỉ copy vào `docs/audit/tickets/` khi nhóm duyệt.

## Đọc gì khi họp

| Ưu tiên | File | Dùng khi |
|---|---|---|
| **1 — mang vào họp** | [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md) | Trình bày & xin duyệt phương án |
| **2 — nếu bị hỏi “sao không X?”** | [02-phu-luc-so-sanh.md](./02-phu-luc-so-sanh.md) | So sánh + cost |
| **3 — backlog / Jira** | [03-backlog.md](./03-backlog.md) · [04-jira-tasks-snippet.md](./04-jira-tasks-snippet.md) | Stories + đoạn dán `JIRA_TASKS.md` |
| **4 — ticket format repo** | [tickets/AUDIT-014-…](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md) · [tickets/AUDIT-015-…](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md) | Copy vào repo khi duyệt |

## Quyết định đề xuất (một dòng)

**Athena + CloudWatch Logs Insights** trên **S3 Object Lock** (giữ SoT).  
Không ELK / Datadog; OpenSearch ingest audit = **P2**.

## Cách dùng 5 phút

1. Mở `01` → §1 + §9 + §10.
2. Phản biện → `02`.
3. Nhóm OK → gửi `tickets/AUDIT-015` cho CDO08, CDO07 làm `AUDIT-014`; dán `04-jira-tasks-snippet.md` vào `JIRA_TASKS.md` nếu cần.

## Nguồn trong repo TF4 (chỉ đọc)

- Evidence: `docs/evidence/mandate-04-forensic/`
- Playbook: `docs/audit/runbooks/forensic-playbook-timeline.md`
- Task sẵn: `docs/audit/DELEGATED_TASKS_P0.md` (3.1 ISM, 3.2 Grafana Audit Dash)
- Format ticket mẫu: `docs/audit/tickets/AUDIT-010-*.md`, `AUDIT-013-*.md`
