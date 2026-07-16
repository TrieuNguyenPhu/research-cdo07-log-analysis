# Backlog đề xuất — sau khi nhóm duyệt phương án

> Ticket format repo (đang ở research, **chưa** vào `tf4-phase3-repo`):
> - [tickets/AUDIT-014-enable-athena-insights-log-analysis.md](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md)
> - [tickets/AUDIT-015-request-athena-glue-audit-permissions.md](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md)
> - Đoạn dán Jira: [04-jira-tasks-snippet.md](./04-jira-tasks-snippet.md)
> - DDL: [05-architecture-athena-forensics.md](./05-architecture-athena-forensics.md)
>
> Research so sánh coi như **done** khi nhóm accept [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md) §10.

---

## Epic

**`[CDO07] Audit Log Analysis Layer — Athena + Insights`**

**Mục tiêu:** Auditor dựng WHO/WHAT/WHEN/HOW **không cần** `aws s3 cp`; S3 Object Lock vẫn là SoT; cost lớp phân tích ≤ $5/tuần.

**Out of scope:** Thay CloudTrail / Firehose / Object Lock; làm lại app logging CDO08; ELK/Datadog; SageMaker / Slack bot (hoãn).

**Gắn sẵn trong repo:**

- Task 3.1 ISM · Task 3.2 Grafana Audit Dash — `docs/audit/DELEGATED_TASKS_P0.md`
- Member 7 / 8 — `docs/audit/TEAM_ASSIGNMENT.md`

**Chuẩn đặt tên (đồng bộ toàn bộ research):**

| Mục | Giá trị |
|---|---|
| Glue DB | `tf4_audit_forensics` |
| Workgroup | `tf4-cdo07-audit` |
| Result | `s3://tf4-athena-query-results-511825856493/cdo07/` |
| IAM owner | **CDO08** (AUDIT-015) |

---

## Stories (copy Jira)

### Done (research — đóng khi họp duyệt)

| ID | Summary | Note |
|---|---|---|
| LOG-01 | Phân tích hiện trạng + pain forensic | `01` §2 |
| LOG-02 | So sánh OpenSearch / ELK / Loki / Datadog / Athena | `01` + `02` |
| LOG-03 | Quyết định phương án | `01` §10 |

---

### P0 — PoC & docs

#### LOG-04b = AUDIT-015 — IAM Athena/Glue/Insights read-only
- **Type:** Task · **Estimate:** 0.5 SP  
- **Assignee:** **CDO08** (SSO/IAM)  
- **DoD:** Theo [AUDIT-015](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md); không `s3:DeleteObject` trên audit SoT buckets

#### LOG-04 = AUDIT-014.1 — PoC Athena
- **Type:** Spike · **Estimate:** 2 SP  
- **Assignee:** CDO07 · **Support:** CDO04 (IaC workgroup/Glue nếu cần)  
- **Depends on:** LOG-04b / AUDIT-015  
- **DoD:**
  - [ ] Glue DB `tf4_audit_forensics` + table `cloudtrail_events` (partition projection)
  - [ ] Workgroup `tf4-cdo07-audit` + limit bytes scanned
  - [ ] (Nếu kịp) `eks_audit_events`
  - [ ] ≥3 query map AUD-17.2; không `aws s3 cp`
  - [ ] Bảng thời gian vs CLI + DataScanned / cost
  - [ ] DDL theo [05](./05-architecture-athena-forensics.md)

#### LOG-05 = AUDIT-014.3 — Sửa drift playbook forensic
- **Type:** Docs · **Estimate:** 0.25 SP  
- **File:** `docs/audit/runbooks/forensic-playbook-timeline.md`  
- **DoD:** Bỏ `LogFileValidationEnabled=false`; Object Lock + validate-logs; gợi ý Insights/Athena

---

### P1 — Drill gần & UI

#### LOG-06 = AUDIT-014.2 — Saved CloudWatch Logs Insights
- **Type:** Spike · **Estimate:** 1 SP  
- **DoD:**
  - [ ] ≥2 query `/aws/eks/techx-tf4-cluster/cluster`
  - [ ] ≥1 query `/aws/cloudtrail/tf4-general-cloudtrail`
  - [ ] Document trong playbook

#### LOG-07 = AUDIT-014.4 — Extend Task 3.2 Grafana Audit Dashboard
- **Type:** Story · **Estimate:** 1–2 SP  
- **Assignee:** Member 8 / CDO07  
- **DoD:** panel 401/403 (nếu chưa) + link Explore Insights; không bắt buộc OpenSearch ingest

#### LOG-08 — Runbook forensic end-to-end
- **Type:** Docs · **Estimate:** 1 SP  
- **DoD:** Insights (gần) → Athena (sâu) → CLI fallback; mentor biết cách xem

#### LOG-04c (optional P1) — Glue table Config
- **DoD:** `aws_config_history` trên staging (ghi rõ staging ≠ WORM SoT)

---

### P2 — Chỉ mở khi điều kiện đủ

#### LOG-09 — OpenSearch index `audit-*` (optional)
- **Điều kiện:** PVC/capacity OK + security plugin + ISM (Task 3.1)  
- **Không làm nếu** disk watermark / plugin tắt

---

### Won’t do (phase này)

| Item | Lý do |
|---|---|
| ELK riêng | Trùng OpenSearch |
| Datadog forensic SoT | Cost + egress |
| Loki primary CloudTrail | Fit kém JSON audit |
| Ingest audit vào OpenSearch ngay | Capacity + security |
| SageMaker / Slack bot / QuickSight | Ngoài scope sprint |

---

## Thứ tự sprint

```
Sau họp duyệt
  AUDIT-015 (CDO08)  →  AUDIT-014 PoC Athena (CDO07)  →  playbook drift
  Insights saved  →  Grafana Task 3.2  →  runbook E2E
  (sau) Config table / OpenSearch audit nếu điều kiện đủ
```

---

## DoD epic

1. ≥1 đường phân tích S3 **không** download thủ công.  
2. Nhóm duyệt loại ELK/Datadog / hoãn OpenSearch ingest.  
3. Drill mẫu đo được thời gian; rõ ràng hơn CLI thuần.  
4. S3 Object Lock vẫn SoT trong runbook.  
5. LOG-09 chỉ mở khi điều kiện P2 thỏa.
