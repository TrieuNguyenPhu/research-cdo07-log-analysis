# Backlog đề xuất — sau khi nhóm duyệt phương án

> Ticket format repo (đang ở research, **chưa** vào `tf4-phase3-repo`):
> - [tickets/AUDIT-014-enable-athena-insights-log-analysis.md](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md)
> - [tickets/AUDIT-015-request-athena-glue-audit-permissions.md](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md)
> - Đoạn dán Jira: [04-jira-tasks-snippet.md](./04-jira-tasks-snippet.md)
>
> Research so sánh coi như **done** khi nhóm accept phương án trong [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md) §10.

---

## Epic

**`[CDO07] Audit Log Analysis Layer — Athena + Insights`**

**Mục tiêu:** Auditor dựng WHO/WHAT/WHEN/HOW **không cần** `aws s3 cp`; S3 Object Lock vẫn là SoT; cost lớp phân tích ≤ $5/tuần.

**Out of scope:** Thay CloudTrail / Firehose / Object Lock; làm lại app logging CDO08; ELK/Datadog.

**Gắn sẵn trong repo:**

- Task 3.1 ISM · Task 3.2 Grafana Audit Dash — `docs/audit/DELEGATED_TASKS_P0.md`
- Member 7 / 8 — `docs/audit/TEAM_ASSIGNMENT.md`

---

## Stories (copy Jira)

### Done (research — đóng khi họp duyệt)

| ID | Summary | Note |
|---|---|---|
| LOG-01 | Phân tích hiện trạng + pain forensic | Covered trong file `01` §2 |
| LOG-02 | So sánh OpenSearch / ELK / Loki / Datadog / Athena | Covered trong `01` + `02` |
| LOG-03 | Quyết định phương án | Checklist `01` §10 |

---

### P0 — PoC & docs

#### LOG-04 — PoC Amazon Athena trên S3 CloudTrail Object Lock
- **Type:** Spike · **Estimate:** 2 SP  
- **Assignee:** CDO07 · **Support:** CDO04 (IAM)  
- **DoD:**
  - [ ] Glue table / partition projection → bucket `tf4-cloudtrail-logs-bucket-511825856493`
  - [ ] Workgroup `tf4-cdo07-audit` + limit bytes scanned
  - [ ] ≥3 query map scenario AUD-17.2 (hoặc tương đương WHO/WHAT/WHEN)
  - [ ] Không dùng `aws s3 cp` trong quy trình PoC
  - [ ] Bảng thời gian: Athena vs CLI
  - [ ] Ghi cost thực tế (Data scanned)

#### LOG-04b — Ticket IAM Athena/Glue read-only
- **Type:** Task · **Estimate:** 0.5 SP  
- **DoD:** Permission set `TF4-AuditReadOnlyAndAnalyze` có quyền đọc Athena/Glue/S3 GetObject cần thiết; không có delete

#### LOG-05 — Sửa drift playbook forensic
- **Type:** Docs · **Estimate:** 0.25 SP  
- **File:** `docs/audit/runbooks/forensic-playbook-timeline.md`  
- **DoD:** Bỏ đoạn `LogFileValidationEnabled=false`; cập nhật Object Lock + validate-logs; gợi ý Insights/Athena

---

### P1 — Drill gần & UI

#### LOG-06 — Saved CloudWatch Logs Insights (EKS + CloudTrail)
- **Type:** Spike · **Estimate:** 1 SP  
- **DoD:**
  - [ ] ≥2 query trên `/aws/eks/techx-tf4-cluster/cluster`
  - [ ] ≥1 query trên `/aws/cloudtrail/tf4-general-cloudtrail`
  - [ ] Document trong playbook / wiki nhóm
  - [ ] So thời gian với CLI

#### LOG-07 — Extend Task 3.2 Grafana Audit Dashboard
- **Type:** Story · **Estimate:** 1–2 SP  
- **Link:** DELEGATED 3.2 / JIRA Task 8 / Member 8  
- **DoD hiện có:** panel 401/403  
- **DoD mở rộng:**
  - [ ] Link/Explore tới Insights forensic
  - [ ] Không bắt buộc ingest OpenSearch

#### LOG-08 — Cập nhật runbook forensic end-to-end
- **Type:** Docs · **Estimate:** 1 SP  
- **DoD:** Playbook: Insights (gần) → Athena (sâu) → CLI fallback; mentor biết cách xem

---

### P2 — Chỉ mở khi điều kiện đủ

#### LOG-09 — OpenSearch index `audit-*` (optional)
- **Điều kiện mở:** PVC/capacity ổn (vd. ≥20Gi hoặc tương đương) + security plugin bật + ISM (Task 3.1)  
- **DoD:** Index tách `otel-logs-*`; ACL audit-only; runbook “lệch thì tin S3”  
- **Không làm nếu** disk watermark / plugin vẫn tắt

---

### Won’t do (phase này)

| Item | Lý do |
|---|---|
| ELK riêng | Trùng OpenSearch |
| Datadog forensic SoT | Cost + egress |
| Loki primary CloudTrail | Fit kém JSON audit |
| Ingest audit vào OpenSearch ngay | Capacity + security |

---

## Thứ tự sprint gợi ý

```
Sau họp duyệt
  LOG-04b IAM  →  LOG-04 PoC Athena  →  LOG-05 playbook drift
  LOG-06 Insights  →  LOG-07 Grafana Task 3.2  →  LOG-08 runbook
  (sau) LOG-09 nếu CDO08 sẵn sàng
```

---

## DoD epic

1. ≥1 đường phân tích S3 **không** download thủ công.  
2. Nhóm đã duyệt loại ELK/Datadog / hoãn OpenSearch ingest.  
3. Drill mẫu đo được thời gian; hướng dưới 10 phút rõ ràng hơn CLI thuần.  
4. S3 Object Lock vẫn SoT trong tài liệu/runbook.  
5. LOG-09 chỉ mở khi điều kiện P2 thỏa.
