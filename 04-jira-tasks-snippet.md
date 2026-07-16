# Đoạn đề xuất gắn vào `docs/audit/JIRA_TASKS.md` (khi nhóm duyệt)

> **Chưa merge vào repo.** Dán vào `JIRA_TASKS.md` sau Task 18 / trước mục “Quy tắc Cập nhật”, khi CDO07 chốt đưa backlog chính thức.  
> Ticket chi tiết: `tickets/AUDIT-014-*.md`, `tickets/AUDIT-015-*.md` trong thư mục research này.

---

## 🔍 Audit Log Analysis Layer (sau Mandate-04)

> Pain còn lại sau Mandate-04: log rải nhiều chỗ, audit sâu phải download S3 + CLI/jq.  
> Phương án: **Athena + CloudWatch Logs Insights** trên SoT S3 Object Lock (không ELK/Datadog; OpenSearch ingest = P2).

**Epic: AUDIT-014 — Athena + Insights Log Analysis**
- **Issue Type:** Epic / Task parent
- **Summary:** `[CDO07] Triển khai lớp phân tích log audit — Athena + CloudWatch Logs Insights`
- **Assignee:** Trieu Nguyen + CDO07
- **Label:** `Lab`, `Need-Review`, `P1`
- **Priority:** P1
- **Estimate:** ~5 SP (PoC + Insights + docs; chưa gồm P2 OpenSearch)
- **Mục tiêu:** Auditor dựng WHO/WHAT/WHEN/HOW không cần `aws s3 cp`; cost lớp phân tích ≤ $5/tuần; S3 Object Lock vẫn SoT.
- **Ticket:** [AUDIT-014-enable-athena-insights-log-analysis.md](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md)

**Task 25: [CDO08] IAM Athena/Glue/Insights read-only (AUDIT-015) — DELEGATED**
- **Issue Type:** Task
- **Summary:** `[CDO07→CDO08] Bổ sung quyền Athena/Glue/Insights cho TF4-AuditReadOnlyAndAnalyze`
- **Assignee:** CDO08 (SSO/IAM)
- **Label:** `Need-Review`, `P0`, `Blocker`
- **Priority:** P0 (blocker AUDIT-014)
- **Estimate:** 0.5–1 SP
- **Mục tiêu:** CDO07 query Athena + Insights được; không có quyền delete trên audit SoT buckets.
- **Ticket:** [AUDIT-015-request-athena-glue-audit-permissions.md](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md)
**✅ Sub-tasks (Definition of Done):**
  - [ ] Permission set cập nhật theo policy trong AUDIT-015.
  - [ ] CDO07 hết AccessDenied Athena/Glue/Insights cần thiết.
  - [ ] Xác nhận không cấp `s3:DeleteObject` trên CloudTrail/EKS audit buckets.

**Task 26: [CDO07] PoC Athena trên CloudTrail S3 (AUDIT-014.1)**
- **Issue Type:** Spike
- **Summary:** `[CDO07] PoC Athena — query CloudTrail Object Lock không download S3`
- **Assignee:** CDO07
- **Label:** `Lab`, `P0`
- **Priority:** P0
- **Estimate:** 2 SP
- **Depends on:** Task 25 (AUDIT-015)
**✅ Sub-tasks (Definition of Done):**
  - [ ] Workgroup `tf4-cdo07-audit` + Glue table CloudTrail.
  - [ ] ≥3 query map forensic; không `aws s3 cp`.
  - [ ] Bảng thời gian vs CLI + data scanned / cost.
  - [ ] **Evidence:** dưới `docs/evidence/` theo AUDIT-014.

**Task 27: [CDO07] Saved Logs Insights EKS + CloudTrail (AUDIT-014.2)**
- **Issue Type:** Spike
- **Summary:** `[CDO07] Saved CloudWatch Logs Insights cho forensic drill gần`
- **Assignee:** CDO07
- **Label:** `Lab`, `P1`
- **Priority:** P1
- **Estimate:** 1 SP
**✅ Sub-tasks (Definition of Done):**
  - [ ] ≥2 query `/aws/eks/techx-tf4-cluster/cluster`.
  - [ ] ≥1 query `/aws/cloudtrail/tf4-general-cloudtrail`.
  - [ ] Document trong playbook.

**Task 28: [CDO07] Cập nhật forensic playbook + sửa drift validation (AUDIT-014.3)**
- **Issue Type:** Docs
- **Summary:** `[CDO07] Playbook forensic: Insights/Athena + sửa LogFileValidationEnabled drift`
- **Assignee:** CDO07
- **Label:** `Docs`, `P0`
- **Priority:** P0
- **Estimate:** 0.5 SP
**✅ Sub-tasks (Definition of Done):**
  - [ ] Bỏ đoạn playbook còn ghi `LogFileValidationEnabled=false`.
  - [ ] Mô tả Insights (gần) → Athena (sâu) → CLI fallback.
  - [ ] Khẳng định S3 Object Lock là SoT.

**Task 29: [CDO07] Extend Task 3.2 Grafana Audit Dashboard (AUDIT-014.4)**
- **Issue Type:** Story
- **Summary:** `[CDO07] Grafana Audit Dash — link Insights forensic (extend Task 3.2)`
- **Assignee:** Member 8 / CDO07
- **Label:** `Lab`, `P1`
- **Priority:** P1
- **Estimate:** 1–2 SP
- **Related:** `DELEGATED_TASKS_P0.md` Task 3.2
**✅ Sub-tasks (Definition of Done):**
  - [ ] Panel 401/403 theo plan Task 3.2 (nếu chưa xong).
  - [ ] Link/Explore tới Insights forensic.
  - [ ] Không bắt buộc ingest OpenSearch.
