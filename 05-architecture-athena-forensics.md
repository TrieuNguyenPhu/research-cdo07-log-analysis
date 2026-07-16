# Architecture & Implementation Proposal — Athena + Insights (bản chỉnh)

| | |
|---|---|
| **Dự án** | Task Force 4 · XBrain |
| **Chủ đề** | Amazon Athena (+ CloudWatch Logs Insights) cho phân tích forensic trên audit trail |
| **Owner** | CDO-07 (Auditability) |
| **Phối hợp** | CDO-08 (IAM/SSO) · CDO-04 (Platform/IaC nếu cần) |
| **Trạng thái** | Đề xuất triển khai · Version 1.1 |
| **Ngày** | 2026-07-16 · Version 1.1 |
| **Account / Region** | `511825856493` / `us-east-1` |

> Tóm tắt họp ngắn: [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md)  
> Ticket: [AUDIT-014](./tickets/AUDIT-014-enable-athena-insights-log-analysis.md) · [AUDIT-015](./tickets/AUDIT-015-request-athena-glue-audit-permissions.md)

---

## I. Bối cảnh & động lực

Directive #4 yêu cầu dựng WHO/WHAT/WHEN/HOW từ audit trail trong thời gian giới hạn, và chứng minh log đáng tin. Mandate-04 đã xong lớp **ghi + khóa** (Object Lock, Firehose, validate-logs, drill 3/3 PASS bằng CLI).

**Pain còn lại:** đọc `.json.gz` trên S3 bằng download + script; CLI/jq sát 10 phút; nhiều nguồn.

### Nguồn dữ liệu (đã có — không thay)

| # | Nguồn | Bucket / path (account `511825856493`) | Ghi chú |
|---|---|---|---|
| 1 | CloudTrail | `s3://tf4-cloudtrail-logs-bucket-511825856493/AWSLogs/511825856493/CloudTrail/` | Object Lock COMPLIANCE 90d · path `region/yyyy/MM/dd/` |
| 2 | AWS Config | Staging: `tf4-aws-config-staging-511825856493-us-east-1/aws-config/` · WORM: `tf4-aws-config-worm-archive-511825856493-us-east-1` | Staging **không** default Object Lock; WORM = SoT dài hạn |
| 3 | EKS audit | `s3://tf4-eks-audit-logs-511825856493/year=…/month=…/day=…/hour=…/` | Firehose · Object Lock COMPLIANCE 90d |

**Cửa sổ gần (không cần Athena):**

- `/aws/cloudtrail/tf4-general-cloudtrail`
- `/aws/eks/techx-tf4-cluster/cluster`  
→ **CloudWatch Logs Insights** cho drill mentor vài giờ–vài ngày.

### Vì sao Athena (và không chỉ Athena)

| Nhu cầu | Công cụ |
|---|---|
| Audit sâu / JOIN nhiều ngày / không download S3 | **Athena** |
| Drill gần ≤10 phút trên CWL | **Logs Insights** |
| Chứng minh không sửa/xóa | **S3 Object Lock + validate-logs** (không dùng Athena làm SoT) |

Athena tận dụng GZIP đã có, scan theo partition, serverless — **không** phải “forensics real-time streaming”.

---

## II. Kiến trúc giải pháp

```
S3 Object Lock / Config WORM     = SOURCE OF TRUTH (giữ nguyên)
         │ READ ONLY
         ▼
┌─────────────────────┐     ┌──────────────────────────┐
│ Glue DB             │     │ CloudWatch Logs Insights │
│ tf4_audit_forensics │     │ CT + EKS log groups      │
│ + 3 external tables │     │ (drill gần)              │
└─────────┬───────────┘     └────────────┬─────────────┘
          │ Athena workgroup             │
          │ tf4-cdo07-audit              │
          └──────────────┬───────────────┘
                         ▼
              CDO07 / mentor forensic demo
```

**Thành phần bắt buộc**

| Thành phần | Giá trị |
|---|---|
| Glue database | `tf4_audit_forensics` |
| Athena workgroup | `tf4-cdo07-audit` · limit bytes scanned (vd. 5–10 GB/query) |
| Result bucket | `s3://tf4-athena-query-results-511825856493/cdo07/` · lifecycle 7–30 ngày · **không** Object Lock |
| IAM | Permission set `TF4-AuditReadOnlyAndAnalyze` — xem AUDIT-015 |

---

## III. Glue tables (DDL chỉnh theo path thật)

> DDL dưới đây là **điểm khởi đầu PoC**. CloudTrail dùng **partition projection** vì path không có `year=` keys. EKS dùng Hive-style từ Firehose — `MSCK REPAIR` hoặc projection đều được.

### 1. `cloudtrail_events` (ưu tiên PoC đầu)

```sql
-- Database
CREATE DATABASE IF NOT EXISTS tf4_audit_forensics;

-- CloudTrail: path .../CloudTrail/us-east-1/2026/07/15/...
-- Dùng projection thay vì MSCK trên Hive keys
CREATE EXTERNAL TABLE tf4_audit_forensics.cloudtrail_events (
  eventVersion STRING,
  userIdentity STRUCT<
    type:STRING,
    principalId:STRING,
    arn:STRING,
    accountId:STRING,
    invokedBy:STRING,
    accessKeyId:STRING,
    userName:STRING,
    sessionContext:STRUCT<
      attributes:STRUCT<mfaAuthenticated:STRING, creationDate:STRING>,
      sessionIssuer:STRUCT<type:STRING, principalId:STRING, arn:STRING, accountId:STRING, userName:STRING>
    >
  >,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  awsRegion STRING,
  sourceIPAddress STRING,
  userAgent STRING,
  errorCode STRING,
  errorMessage STRING,
  requestParameters STRING,
  responseElements STRING,
  additionalEventData STRING,
  requestId STRING,
  eventId STRING,
  readOnly STRING,
  resources ARRAY<STRUCT<arn:STRING, accountId:STRING, type:STRING>>,
  eventType STRING,
  recipientAccountId STRING,
  sharedEventID STRING,
  vpcEndpointId STRING
)
PARTITIONED BY (
  region STRING,
  year STRING,
  month STRING,
  day STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://tf4-cloudtrail-logs-bucket-511825856493/AWSLogs/511825856493/CloudTrail/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.region.type' = 'enum',
  'projection.region.values' = 'us-east-1',
  'projection.year.type' = 'integer',
  'projection.year.range' = '2026,2030',
  'projection.month.type' = 'integer',
  'projection.month.range' = '01,12',
  'projection.month.digits' = '2',
  'projection.day.type' = 'integer',
  'projection.day.range' = '01,31',
  'projection.day.digits' = '2',
  'storage.location.template' = 's3://tf4-cloudtrail-logs-bucket-511825856493/AWSLogs/511825856493/CloudTrail/${region}/${year}/${month}/${day}'
);
```

> Sau PoC có thể mở rộng `projection.region.values` nếu trail multi-region ghi nhiều region path.

### 2. `aws_config_history` (P1 — sau CloudTrail)

```sql
-- Khởi đầu trỏ STAGING (gần). SoT dài hạn: đổi LOCATION sang worm-archive khi sẵn sàng.
CREATE EXTERNAL TABLE tf4_audit_forensics.aws_config_history (
  fileVersion STRING,
  configSnapshotId STRING,
  configurationItems ARRAY<STRUCT<
    configurationItemVersion:STRING,
    configurationItemCaptureTime:STRING,
    configurationItemStatus:STRING,
    resourceType:STRING,
    resourceId:STRING,
    resourceName:STRING,
    ARN:STRING,
    awsRegion:STRING,
    availabilityZone:STRING,
    configuration:STRING,
    supplementaryConfiguration:STRING,
    tags:MAP<STRING,STRING>
  >>
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://tf4-aws-config-staging-511825856493-us-east-1/aws-config/'
TBLPROPERTIES ('has_encrypted_data'='false');
```

> Schema Config thực tế có thể cần chỉnh sau khi sample 1 object. Partition theo path Config (`AWSLogs/.../Config/...`) — PoC đo path thật rồi thêm projection tương tự CloudTrail.

### 3. `eks_audit_events` (P0/P1 — partition Hive sẵn)

```sql
CREATE EXTERNAL TABLE tf4_audit_forensics.eks_audit_events (
  kind STRING,
  apiVersion STRING,
  level STRING,
  auditID STRING,
  stage STRING,
  requestURI STRING,
  verb STRING,
  user STRUCT<
    username:STRING,
    uid:STRING,
    groups:ARRAY<STRING>
  >,
  sourceIPs ARRAY<STRING>,
  userAgent STRING,
  objectRef STRUCT<
    resource:STRING,
    namespace:STRING,
    name:STRING,
    uid:STRING,
    apiGroup:STRING,
    apiVersion:STRING
  >,
  responseStatus STRUCT<code:INT, status:STRING, message:STRING, reason:STRING>,
  requestReceivedTimestamp STRING,
  stageTimestamp STRING
)
PARTITIONED BY (
  year STRING,
  month STRING,
  day STRING,
  hour STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://tf4-eks-audit-logs-511825856493/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.year.type' = 'integer',
  'projection.year.range' = '2026,2030',
  'projection.month.type' = 'integer',
  'projection.month.range' = '01,12',
  'projection.month.digits' = '2',
  'projection.day.type' = 'integer',
  'projection.day.range' = '01,31',
  'projection.day.digits' = '2',
  'projection.hour.type' = 'integer',
  'projection.hour.range' = '00,23',
  'projection.hour.digits' = '2',
  'storage.location.template' = 's3://tf4-eks-audit-logs-511825856493/year=${year}/month=${month}/day=${day}/hour=${hour}'
);
```

> Firehose ghi `.gz` nhiều event/line — nếu SerDe lỗi, PoC thử `JsonSerDe` + compression hoặc pipeline transform; ghi rõ gap trong evidence.

---

## IV. Query templates (map forensic drill)

### Scenario A — On-call / bastion (CloudTrail) · AUD-17.2 style

```sql
SELECT
  eventTime,
  eventName,
  userIdentity.arn,
  userIdentity.sessionContext.sessionIssuer.userName AS role_name,
  sourceIPAddress,
  userAgent
FROM tf4_audit_forensics.cloudtrail_events
WHERE eventName = 'StartSession'
  AND year = '2026' AND month = '07' AND day = '15'
ORDER BY eventTime DESC
LIMIT 50;
```

### Scenario B — K8s port-forward / ConfigMap (EKS)

```sql
SELECT
  requestReceivedTimestamp,
  user.username,
  sourceIPs[1] AS source_ip,
  verb,
  objectRef.resource,
  objectRef.name,
  objectRef.namespace,
  responseStatus.code
FROM tf4_audit_forensics.eks_audit_events
WHERE verb IN ('create', 'update', 'patch')
  AND (
    objectRef.resource IN ('pods/portforward', 'configmaps', 'secrets')
    OR objectRef.name LIKE '%flagd%'
  )
  AND year = '2026' AND month = '07' AND day = '15'
  AND hour BETWEEN '07' AND '09'
ORDER BY requestReceivedTimestamp DESC
LIMIT 100;
```

### Scenario C — AccessDenied (CloudTrail)

```sql
SELECT
  eventTime,
  eventName,
  errorCode,
  errorMessage,
  userIdentity.arn,
  sourceIPAddress
FROM tf4_audit_forensics.cloudtrail_events
WHERE errorCode = 'AccessDenied'
  AND year = '2026' AND month = '07' AND day = '15'
ORDER BY eventTime DESC
LIMIT 100;
```

### Scenario D — JOIN CT ↔ Config (P1, minh họa)

Hữu ích cho change-trail (vd. security group) nhưng **không** DoD PoC tuần 1. Làm sau khi schema Config ổn định.

### Insights (song song — drill gần)

```
fields @timestamp, @message
| filter @message like /StartSession/ or @message like /portforward/
| sort @timestamp desc
| limit 50
```

Log groups: `/aws/cloudtrail/tf4-general-cloudtrail`, `/aws/eks/techx-tf4-cluster/cluster`.

---

## V. Chi phí (us-east-1)

| Hạng mục | Rate / giả định |
|---|---|
| Athena | $5 / TB scanned |
| Insights | $0.005 / GB scanned |
| Glue Catalog | Free tier thường đủ PoC nhỏ |
| Pipeline ghi hiện tại | ~$0.64/tuần — **giữ nguyên** |

| Nguồn | Ước lượng | Scan 1 ngày (order of magnitude) |
|---|---|---|
| CloudTrail | ~vài–chục MB/ngày (đo lại khi PoC) | << $0.01 |
| Config | thấp | << $0.01 |
| EKS audit | **~170 MB/ngày** (~5.1 GB/tháng) | ~$0.00085 / ngày data |

**1 investigation** (2–5 ngày, có partition): thường **cents**.  
**Ceiling:** lớp phân tích **≤ $5/tuần**; kỳ vọng **<$1–2/tháng** nếu kỷ luật partition.

**Bắt buộc:** luôn `WHERE year/month/day` (và `hour` với EKS); workgroup limit bytes; không schedule full-scan.

> Không claim “unlimited investigations $2/tháng” trong tài liệu mentor — nói **kỳ vọng thấp nếu filter đúng**, kèm ceiling.

---

## VI. Roadmap (đã cắt scope)

### Phase 1 — PoC (1 tuần) · AUDIT-014 + AUDIT-015

- [ ] IAM read-only Athena/Glue/Insights (CDO08 — AUDIT-015)
- [ ] Workgroup + result bucket (+ Glue DB)
- [ ] Table `cloudtrail_events` + ≥3 query scenario A/C
- [ ] (Nếu kịp) `eks_audit_events` + scenario B
- [ ] Đo DataScanned + thời gian vs CLI; **không** `s3 cp`
- [ ] Sửa playbook drift `LogFileValidationEnabled`

### Phase 2 — Templates & UX (1 tuần)

- [ ] Bộ query map đủ AUD-17.2 (≥5 scenario)
- [ ] Saved Insights + document playbook
- [ ] Table Config (staging) nếu cần change-trail SQL
- [ ] Extend Task 3.2 Grafana (link Explore) — Member 8

### Phase 3 — Hoãn (không commit sprint này)

- Automated Lambda partition (projection đã giảm nhu cầu)
- Slack bot, SageMaker, Kinesis Analytics, QuickSight
- OpenSearch ingest `audit-*` — chỉ khi capacity + security plugin OK

---

## VII. Ranh giới & rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Gọi Athena là “real-time” | Tài liệu / demo dùng đúng: interactive SQL + Insights gần |
| Scan cả bucket | Workgroup limit + partition bắt buộc |
| SerDe EKS/Firehose lệch | PoC sample 1 object; ghi gap |
| Config staging ≠ WORM | Ghi rõ trong runbook |
| Nhầm Athena = SoT | Mentor demo vẫn chỉ Object Lock / validate-logs |

---

## VIII. Phê duyệt

Amazon Athena (+ Insights) biến audit trail trên S3 thành **lớp query** cho forensic, **không** thay kho bất biến. Align Directive #4 usability sau khi lớp ghi/khóa đã PASS.

**Lợi ích (đã chỉnh claim):**

- Giảm mạnh thời gian so với download S3 + script (mục tiêu đưa drill rõ dưới 10 phút)
- Chi phí thấp với volume ~5 GB/tháng + partition
- Serverless, ít vận hành
- SQL quen thuộc cho auditor

| | |
|---|---|
| [ ] Approved | |
| [ ] Rejected | |
| [ ] Requires Modification | |

*Ý kiến reviewer:* _________________________________

---

**CDO-07** · 2026-07-16 · v1.1
