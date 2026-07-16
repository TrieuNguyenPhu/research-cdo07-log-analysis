# [AUDIT-014] Triển khai lớp phân tích log audit — Athena + CloudWatch Logs Insights

**Trạng thái**: TO DO  
**Người yêu cầu (Reporter)**: Trieu Nguyen - Nhóm CDO07 (Audit)  
**Người thực hiện (Assignee)**: Nhóm CDO07 (Auditability)  
**Nhóm phối hợp**: Nhóm CDO04 (Platform — Athena workgroup / Glue / result bucket nếu cần IaC), Nhóm CDO08 (IAM/SSO — xem [AUDIT-015](./AUDIT-015-request-athena-glue-audit-permissions.md))  
**Vị trí file:** research (chưa merge repo) · khi duyệt → copy vào `docs/audit/tickets/`  
**Độ ưu tiên (Priority)**: P1 (Cải thiện forensic usability sau Mandate-04)  
**Epic**: Auditability — Audit Log Analysis Layer (sau Mandate-04)  
**Account / Region**: `511825856493` / `us-east-1`

---

## 1. Bối cảnh (Context)

Mandate-04 (Directive #4) đã hoàn thành lớp **ghi + khóa** audit trail:

- CloudTrail `tf4-general-cloudtrail` → S3 Object Lock COMPLIANCE 90 ngày + CloudWatch Logs `/aws/cloudtrail/tf4-general-cloudtrail`.
- EKS audit → CloudWatch `/aws/eks/techx-tf4-cluster/cluster` → Firehose `tf4-eks-audit-logs-firehose` → S3 Object Lock COMPLIANCE 90 ngày.
- Forensic drill AUD-17.2: **3/3 PASS** (≤10 phút) nhưng quy trình chủ yếu **CLI + jq**, sát hạn; audit sâu trên S3 vẫn phải `aws s3 cp` + script.

Theo phân công trụ:

- `docs/epic-01-addressing-system-gap/GAP-TO-PILLAR-MAPPING.md`: CDO07 owns Auditability (audit log, evidence, change trail).
- `docs/audit/TEAM_ASSIGNMENT.md`: Member 7 (OpenSearch ISM / Task 3.1), Member 8 (Grafana Audit Dashboard / Task 3.2).
- `docs/audit/DELEGATED_TASKS_P0.md`: Task 3.1 ISM, Task 3.2 Grafana Audit Dashboard đã nằm trong kế hoạch.

CDO07 đã nghiên cứu OpenSearch / ELK / Loki / Datadog và chốt hướng Athena trên 3 nguồn S3 (CloudTrail + Config + EKS) kèm Insights. **Phương án:**

> Giữ S3 Object Lock làm source of truth.  
> Thêm lớp đọc: **Amazon Athena** (partition/projection) + **CloudWatch Logs Insights** (drill gần).  
> Không ELK/Datadog; không ingest OpenSearch giai đoạn này; không claim “real-time streaming”.

Chi tiết kiến trúc/DDL: research `05-architecture-athena-forensics.md`.

Ticket này là triển khai / nghiệm thu PoC. Quyền IAM tách **AUDIT-015** (blocker · CDO08).

---

## 2. Yêu cầu (The What)

### 2.1. PoC Amazon Athena (P0)

1. Tạo Athena workgroup `tf4-cdo07-audit` + result prefix `s3://tf4-athena-query-results-511825856493/cdo07/` (lifecycle ngắn, limit bytes scanned).
2. Glue database `tf4_audit_forensics` theo DDL trong research `05-architecture-athena-forensics.md`:
   - **P0:** `cloudtrail_events` (partition projection theo `region/yyyy/MM/dd`)
   - **P0/P1:** `eks_audit_events` (Hive path Firehose `year=/month=/day=/hour=`)
   - **P1:** `aws_config_history` (staging trước; WORM archive khi ổn)
3. ≥3 query map AUD-17.2 (StartSession, AccessDenied, EKS create/portforward) — **không** `aws s3 cp`.
4. Bảng thời gian Athena vs CLI + `DataScannedInBytes` / ước cost.
5. Không làm DoD tuần 1: JOIN cross-service phức tạp, SageMaker, Slack bot.

### 2.2. Saved CloudWatch Logs Insights (P1)

Tạo và document ≥3 saved query:

| # | Log group | Mục đích mẫu |
|---|---|---|
| 1–2 | `/aws/eks/techx-tf4-cluster/cluster` | ConfigMap / portforward / secret-related |
| 3 | `/aws/cloudtrail/tf4-general-cloudtrail` | StartSession hoặc AccessDenied |

So sánh thời gian với `filter-log-events` / `lookup-events` + jq.

### 2.3. Extend Task 3.2 Grafana Audit Dashboard (P1)

Gắn với `DELEGATED_TASKS_P0.md` Task 3.2 / Member 8:

- Giữ panel 401/403 theo plan hiện có.
- Bổ sung link/Explore tới Insights forensic (CloudWatch datasource) — **không** bắt buộc ingest OpenSearch.

### 2.4. Docs / runbook (P0–P1)

1. Sửa drift trong `docs/audit/runbooks/forensic-playbook-timeline.md`: đoạn còn ghi `LogFileValidationEnabled = false` (evidence AUD-17.1/17.3 đã `true`).
2. Cập nhật playbook: Insights (gần) → Athena (sâu) → CLI fallback.
3. Khẳng định trong runbook: **S3 Object Lock vẫn là SoT**; Athena/Insights chỉ là lớp đọc.

### 2.5. Ngoài phạm vi ticket này

- Không tắt/thay Firehose, Object Lock, CloudTrail log validation.
- Không dựng ELK / Datadog làm forensic SoT.
- Không ingest CloudTrail/EKS audit vào OpenSearch (P2 — chỉ mở khi CDO08: capacity + security plugin + ISM Task 3.1).

---

## 3. Ranh giới trách nhiệm

| Nhóm | Ownership |
|---|---|
| **CDO07** | PoC query, evidence, runbook, nghiệm thu cost ceiling, extend yêu cầu Task 3.2 |
| **CDO08** | Cập nhật SSO Permission Set (AUDIT-015); ISM OpenSearch nếu sau này P2 |
| **CDO04** | IaC Athena workgroup / Glue / result bucket nếu team chốt provision bằng Terraform |

---

## 4. Cost guardrail

| Mục | Giá trị |
|---|---|
| Pipeline ghi hiện tại (giữ) | ~$0.64/tuần |
| Ceiling lớp phân tích (Athena + Insights) | **≤ $5/tuần** |
| Kỳ vọng với audit thưa + partition | **<$1/tuần** |

Bắt buộc: workgroup limit bytes scanned; Insights luôn hẹp time window; không scheduled query rộng.

---

## 5. Tiêu chí nghiệm thu (Acceptance Criteria / Evidence)

- [ ] AUDIT-015 completed (không còn AccessDenied Athena/Glue cần thiết).
- [ ] Athena: ≥1 table CloudTrail query được; ≥3 query forensic mẫu chạy thành công **không** `s3 cp`.
- [ ] Có bảng thời gian Athena (và/hoặc Insights) vs CLI cho ≥1 scenario AUD-17.2.
- [ ] Insights: ≥2 query EKS + ≥1 query CloudTrail CWL đã document.
- [ ] Playbook forensic đã sửa drift validation + mô tả Athena/Insights.
- [ ] Cost PoC ghi nhận (Data scanned / Cost Explorer) trong ceiling ≤ $5/tuần.
- [ ] Evidence lưu tại `docs/evidence/` (thư mục đề xuất: `docs/evidence/audit-log-analysis/` hoặc dưới `mandate-04-forensic/` nếu nhóm chốt).
- [ ] Task 3.2: có note/requirement mở rộng Grafana (panel hoặc link Explore) — hoặc ticket follow-up nếu Member 8 tách riêng.

---

## 6. Evidence / lệnh tham chiếu PoC

```sql
-- Athena mẫu: StartSession (bắt buộc filter partition)
SELECT
  eventTime,
  eventName,
  userIdentity.arn AS principal_arn,
  sourceIPAddress
FROM tf4_audit_forensics.cloudtrail_events
WHERE eventName = 'StartSession'
  AND year = '2026' AND month = '07' AND day = '15'
ORDER BY eventTime DESC
LIMIT 50;
```

```text
# Insights — EKS (chỉnh field sau PoC)
fields @timestamp, @message
| filter @message like /portforward/
| sort @timestamp desc
| limit 50
```

Profile: `TF4-AuditReadOnlyAndAnalyze` (sau AUDIT-015).

---

## 7. Related

| Tài liệu | Đường dẫn |
|---|---|
| Mandate-04 evidence | `docs/evidence/mandate-04-forensic/` |
| Forensic framework | `docs/audit/tickets/AUDIT-CDO07-MANDATE04-FORENSIC-FRAMEWORK.md` |
| Playbook | `docs/audit/runbooks/forensic-playbook-timeline.md` |
| IAM blocker | [AUDIT-015-request-athena-glue-audit-permissions.md](./AUDIT-015-request-athena-glue-audit-permissions.md) |
| Grafana Audit Dash (sẵn có) | `docs/audit/DELEGATED_TASKS_P0.md` Task 3.2 |
| Architecture DDL (research) | `../05-architecture-athena-forensics.md` |
| Jira snippet | `../04-jira-tasks-snippet.md` |

---

## 8. Subtasks gợi ý (Jira)

| ID | Summary | Estimate | Priority |
|---|---|---|---|
| AUDIT-014.1 | PoC Athena CloudTrail table + ≥3 query | 2 SP | P0 |
| AUDIT-014.2 | Saved Insights EKS + CloudTrail CWL | 1 SP | P1 |
| AUDIT-014.3 | Sửa playbook drift + mô tả Athena/Insights | 0.5 SP | P0 |
| AUDIT-014.4 | Extend yêu cầu / nghiệm thu Task 3.2 Grafana | 1 SP | P1 |
| AUDIT-014.5 | (Optional P2) OpenSearch `audit-*` — chỉ khi capacity + sec OK | — | P2 |

*(Sau khi hoàn thành, tag Lead CDO07 để review evidence và cost.)*
