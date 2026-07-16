# Giải pháp đề xuất — Lớp phân tích log audit (CDO07)

> **Vị trí tài liệu:** `C:\Users\NguyenPhuTrieu\Downloads\research-cdo07-log-analysis`  
> (ngoài git repo — không ảnh hưởng nghiệm thu Mandate-04)  
> **Quyết định đề xuất:** **Athena (P0) + Grafana/CloudWatch Insights (P1 nhẹ) + OpenSearch ingest audit (P2 tùy chọn)**  
> **Không chọn:** ELK riêng, Datadog làm SoT, Loki làm primary CloudTrail

---

## 1. Vấn đề cần giải (một câu)

Mandate-04 đã **ghi và khóa** log trên S3 Object Lock; CDO07 vẫn phải **tải S3 / CLI + jq** mỗi lần audit → chậm, rải rác, sát 10 phút drill.

---

## 2. Kiến trúc mục tiêu

> Sync repo 16/07: CloudTrail đã có CWL group; OpenSearch PVC 8Gi đang áp lực đĩa — xem `07-sync-repo-*.md`.

```
                    ┌─────────────────────────────────────┐
                    │  SOURCE OF TRUTH (đã có — KHÔNG đụng) │
                    │  S3 Object Lock COMPLIANCE 90 ngày   │
                    │  • tf4-cloudtrail-logs-bucket-*      │
                    │  • tf4-eks-audit-logs-*              │
                    └──────────────┬──────────────────────┘
                                   │ chỉ READ
           ┌───────────────────────┼───────────────────────┐
           ▼                       ▼                       ▼
   ┌───────────────┐     ┌─────────────────┐     ┌──────────────────┐
   │ A. Athena     │     │ B. CWL Insights │     │ C. OpenSearch    │
   │ + Glue table  │     │ EKS + CloudTrail│     │ audit-*  (P2)    │
   │ (query S3)    │     │ log groups      │     │ sau capacity+sec │
   └───────┬───────┘     └────────┬────────┘     └────────┬─────────┘
           │                      │                       │
           └──────────────────────┼───────────────────────┘
                                  ▼
                    + extend Grafana Audit Dash (Task 3.2 sẵn có)
```

| Lớp | Vai trò | Khi dùng |
|---|---|---|
| **S3 Object Lock** | SoT bất biến | Luôn |
| **A. Athena** | > cửa sổ CWL / scan S3 / không download | Audit sâu |
| **B. Insights** | Drill gần: `/aws/eks/.../cluster` **và** `/aws/cloudtrail/tf4-general-cloudtrail` | Mentor / sự kiện trong tuần |
| **C. OpenSearch** | Full-text UI | Chỉ sau PVC≥~20Gi + security plugin + ISM |

---

## 3. Phương án A — Athena (làm trước)

### 3.1. Vì sao đây là giải pháp đúng pain task

| Pain task | Athena giải thế nào |
|---|---|
| Log trên S3 phải down về chạy code | `SELECT` trên `.json.gz` tại chỗ — zero download |
| Nhiều nguồn | 2 bảng SQL: `cloudtrail_logs`, `eks_audit_logs` |
| Budget ~$300/tuần | Pay-per-query; volume ~5 GB/tháng → thường **<$1–5/tháng** nếu ít audit |
| Không phá tamper-evident | Không ghi/xóa object; chỉ đọc |

### 3.2. Thành phần cần tạo (infra — nhờ CDO04 / quyền CDO07)

| Thành phần | Giá trị gợi ý |
|---|---|
| Region | `us-east-1` |
| Athena workgroup | `tf4-cdo07-audit` (result bucket riêng, **không** Object Lock bắt buộc trên result) |
| Query result location | `s3://tf4-athena-query-results-<account>/cdo07/` (lifecycle 7–30 ngày) |
| Glue DB | `tf4_audit` |
| Tables | `cloudtrail_events`, `eks_audit_events` |
| IAM | Bổ sung read-only Athena/Glue/S3 GetObject cho `TF4-AuditReadOnlyAndAnalyze` |

### 3.3. PoC tối thiểu (Definition of Done kỹ thuật)

1. Tạo Glue table (hoặc partition projection) trỏ đúng prefix CloudTrail trên S3.
2. Chạy được 1 query: “StartSession trong 7 ngày” → có `userIdentity` / session name.
3. (Nếu schema EKS sẵn) 1 query: ConfigMap update / portforward trong window drill.
4. Bảng so sánh thời gian: CLI playbook vs Athena (cùng 1 scenario AUD-17.2).
5. Screenshot / ghi chú: **không** dùng `aws s3 cp` trong quy trình mới.

### 3.4. Query mẫu (CloudTrail — chỉnh table/path theo Glue thật)

```sql
-- WHO/WHAT/WHEN: SSM StartSession (on-call / bastion) 7 ngày gần
SELECT
  from_iso8601_timestamp(eventtime) AS event_time,
  eventname,
  useridentity.arn AS principal_arn,
  useridentity.sessioncontext.sessionissuer.username AS role_name,
  sourceipaddress,
  useragent
FROM tf4_audit.cloudtrail_events
WHERE eventname = 'StartSession'
  AND from_iso8601_timestamp(eventtime)
      >= current_timestamp - INTERVAL '7' DAY
ORDER BY event_time DESC
LIMIT 50;
```

```sql
-- Unauthorized: AccessDenied
SELECT
  from_iso8601_timestamp(eventtime) AS event_time,
  eventname,
  errorcode,
  errormessage,
  useridentity.arn AS principal_arn,
  resources
FROM tf4_audit.cloudtrail_events
WHERE errorcode = 'AccessDenied'
  AND from_iso8601_timestamp(eventtime)
      >= current_timestamp - INTERVAL '24' HOUR
ORDER BY event_time DESC
LIMIT 100;
```

> Schema CloudTrail JSON của AWS có sẵn mẫu Glue/Athena trong docs AWS (“CloudTrail logs Athena”). Khi PoC, ưu tiên dùng template chuẩn rồi chỉnh `LOCATION` sang bucket TF4.

### 3.5. Chi phí thô (để đưa backlog)

Giả sử mỗi lần drill scan **≤2 GB** dữ liệu đã partition theo ngày:

- Athena: $5 / TB scanned → 2 GB ≈ **$0.01 / query**
- 20 query/tuần ≈ **$0.20/tuần** (chưa kể S3 GET — không đáng kể)

→ An toàn trong ngân sách; ghi ceiling đề xuất: **≤ $5/tuần** cho lớp phân tích giai đoạn PoC.

---

## 4. Phương án B — CloudWatch Logs Insights / Grafana (P1 — nâng ưu tiên)

CloudTrail **đã** deliver vào CWL (AUD-17.1): `/aws/cloudtrail/tf4-general-cloudtrail`.  
EKS audit: `/aws/eks/techx-tf4-cluster/cluster`.

**Mục tiêu:** thay phần lớn `filter-log-events`/`lookup-events` + jq bằng Insights (hoặc Grafana Explore → CloudWatch). Playbook đã gợi ý Insights khi lookup chậm — chuẩn hóa thành saved query.

Ví dụ Logs Insights (EKS):

```
fields @timestamp, @message
| filter @message like /portforward/
| sort @timestamp desc
| limit 50
```

(Parse JSON field path theo event thật sau PoC.)

**Ưu:** Zero infra mới; cover **cả** CloudTrail gần + EKS.  
**Nhược:** Không thay Athena cho audit sâu trên S3 Object Lock / ngoài retention CWL.

Gắn **Task 3.2 Grafana Audit Dashboard** (đã có trong DELEGATED/JIRA): mở rộng panel 401/403 + link sang Insights forensic.

---

## 5. Phương án C — OpenSearch (P2 — blocker capacity + security)

Chỉ mở khi:

1. OpenSearch **security plugin bật** (ADR-009 vẫn ghi tắt).
2. **Capacity ổn:** PVC đủ (CDO08 đề xuất 20Gi sau watermark 8Gi) + ISM (Task 3.1).
3. Index `audit-*` tách `otel-logs-*`; runbook: lệch → tin S3 + Athena.

**Không** ingest audit vào OpenSearch khi disk đang watermark hoặc plugin tắt.  
**Không** coi OpenSearch là bằng chứng tamper-evident.

---

## 6. Các giải pháp thị trường — quyết định rõ ràng

| Giải pháp | Quyết định | Lý do ngắn |
|---|---|---|
| **OpenSearch** | P2 tùy chọn (sau harden) | Đã có trong cluster; UI tốt; chưa đủ bảo mật cho audit |
| **ELK** | **Loại** | Trùng năng lực OpenSearch; thêm stack = phí vận hành |
| **Loki** | **Hoãn** | Rẻ cho app log; kém fit JSON CloudTrail lớn trừ khi parse nặng |
| **Datadog** | **Loại** (phase này) | Cost ingest + data rời account; SoT phải ở S3 |
| **Athena** | **Chọn P0** | Đúng pain download S3; rẻ; giữ Object Lock |

---

## 7. Lộ trình triển khai đề xuất

| Tuần | Việc | Owner gợi ý | Output |
|---|---|---|---|
| W1 | Chốt ADR (`05`) + sync repo (`07`) + trình bày | CDO07 | Decision |
| W1 | Sửa drift playbook (LOG-03b) | CDO07 | PR docs |
| W1–W2 | IAM Athena/Glue read-only | CDO07 → CDO04 | Permission set |
| W2 | PoC Athena + Insights (CT CWL + EKS) | CDO07 | Bảng thời gian |
| W2 | Extend yêu cầu Task 3.2 Grafana Audit Dash | Member 8 / CDO07 | Requirement + nghiệm thu |
| Sau | OpenSearch audit ingest | CDO08 + Member 7 ISM | Chỉ khi capacity+sec OK |

---

## 8. Rủi ro & cách giảm

| Rủi ro | Giảm |
|---|---|
| Schema Glue sai → query trống | Dùng template CloudTrail Athena chuẩn; validate bằng 1 event đã biết từ drill |
| Scan cả bucket không partition → bill bất ngờ | Bắt buộc filter theo `year/month/day` hoặc partition projection; workgroup limit bytes scanned |
| Mentor bắt demo CLI raw | Giữ playbook CLI làm fallback; Athena là đường “audit dễ hơn” |
| Nhầm OpenSearch = SoT | ADR + runbook in đậm: SoT = S3 Object Lock |
| Thiếu quyền | Ticket kiểu AUDIT-010 bổ sung (không tự mở quyền rộng) |

---

## 9. Tiêu chí thành công (đo được)

1. Auditor hoàn thành **ít nhất 1** scenario kiểu AUD-17.2 **không** dùng `aws s3 cp`.
2. Thời gian trung bình drill (cùng scenario) **giảm** so với baseline CLI (ghi số phút).
3. Chi phí lớp phân tích **≤ $5/tuần** trong PoC (hoặc số đã ghi trong ADR).
4. Tài liệu ADR + backlog story đã sẵn để paste Jira.

---

## 10. Việc CDO07 làm ngay (không cần chờ infra)

- [x] Bản đồ Mandate + hiện trạng + so sánh (file `00`–`03`)
- [x] Giải pháp đề xuất (file này)
- [ ] Xin slot 15 phút CDO04 (IAM/Athena) + CDO08 (Grafana/OpenSearch sec)
- [ ] Chốt chữ ký ADR file `05`
- [ ] Paste stories từ `03` vào Jira sau khi ADR OK
