# [AUDIT-015] Yêu cầu bổ sung quyền read-only Athena / Glue phục vụ phân tích log audit

**Trạng thái**: TO DO  
**Người yêu cầu (Reporter)**: Trieu Nguyen - Nhóm CDO07 (Audit)  
**Người thực hiện (Assignee)**: Nhóm CDO08 (Security/SSO/IAM)  
**Nhóm phối hợp**: Nhóm CDO07 (nghiệm thu PoC), Nhóm CDO04 (Platform — nếu provision Athena workgroup/Glue bằng Terraform)  
**Độ ưu tiên (Priority)**: P0 (Blocker PoC [AUDIT-014](./AUDIT-014-enable-athena-insights-log-analysis.md))  
**Epic**: Auditability — Audit Log Analysis Layer (sau Mandate-04)  
**Account / Region**: `511825856493` / `us-east-1`  
**Vị trí file:** research (chưa merge repo) · khi duyệt → copy vào `docs/audit/tickets/`

---

## 1. Bối cảnh (Context)

Sau Mandate-04, audit trail bất biến đã nằm trên S3 Object Lock. CDO07 đề xuất lớp phân tích **Amazon Athena** để query thẳng file log trên S3 — giải pain phải `aws s3 cp` + script mỗi lần audit sâu (xem AUDIT-014).

Theo phân công trụ:

- CDO07 chỉ audit/evidence — **không** tự sửa IAM/SSO.
- CDO08 owns Security, access control, least-privilege (`docs/audit/TEAM_ASSIGNMENT.md`, GAP-TO-PILLAR-MAPPING).

Permission set hiện dùng: `TF4-AuditReadOnlyAndAnalyze`.  
Ticket trước (AUDIT-010, AUDIT-013) đã bổ sung quyền forensic CloudTrail/S3/Firehose/Object Lock — **chưa** bao phủ Athena / Glue / result bucket.

Ticket này chỉ yêu cầu quyền **read-only / query** phục vụ PoC; không cấp quyền xóa object audit, không cấp quyền sửa Object Lock.

---

## 2. Yêu cầu từ CDO08 (The What)

Cập nhật SSO Permission Set `TF4-AuditReadOnlyAndAnalyze` với các quyền sau (least privilege, scope resource khi được).

### 2.1. Amazon Athena

| Quyền | Mục đích |
|---|---|
| `athena:StartQueryExecution` | Chạy SQL trên table audit |
| `athena:GetQueryExecution` | Theo dõi trạng thái query |
| `athena:GetQueryResults` | Đọc kết quả |
| `athena:StopQueryExecution` | Hủy query nếu scan quá lớn |
| `athena:ListWorkGroups` | Xác nhận workgroup `tf4-cdo07-audit` |
| `athena:GetWorkGroup` | Kiểm tra config workgroup / limit |
| `athena:ListQueryExecutions` | Lịch sử query phục vụ evidence cost |

### 2.2. AWS Glue Data Catalog

| Quyền | Mục đích |
|---|---|
| `glue:GetDatabase` | Đọc DB `tf4_audit` |
| `glue:GetDatabases` | Liệt kê DB |
| `glue:GetTable` / `glue:GetTables` | Đọc schema `cloudtrail_events` (và EKS nếu có) |
| `glue:GetPartition` / `glue:GetPartitions` | Partition theo ngày (nếu dùng) |

> Nếu CDO04 tạo table bằng role khác: CDO07 **không** cần `glue:Create*` / `glue:Update*` / `glue:Delete*`.  
> Chỉ khi CDO07 phải tự tạo table tạm trong PoC mới cân nhắc `glue:CreateTable` / `glue:CreateDatabase` scoped — ưu tiên CDO04 provision trước.

### 2.3. S3 — đọc audit log + ghi result Athena

| Quyền | Resource gợi ý | Mục đích |
|---|---|---|
| `s3:ListBucket` | `tf4-cloudtrail-logs-bucket-*`, `tf4-eks-audit-logs-*` | Athena list objects |
| `s3:GetObject` | cùng bucket `/*` | Athena đọc `.json.gz` |
| `s3:GetBucketLocation` | cùng bucket | Athena/Glue metadata |
| `s3:ListBucket` + `s3:GetObject` + `s3:PutObject` + `s3:AbortMultipartUpload` | **chỉ** `tf4-athena-query-results-* /cdo07/*` | Athena ghi/đọc query result |
| `s3:GetBucketObjectLockConfiguration` (đã có từ AUDIT-013 nếu rồi) | audit buckets | Không đổi — chỉ xác nhận SoT |

**Cấm / không cấp:** `s3:DeleteObject`, `s3:PutObject` trên CloudTrail/EKS audit buckets, `s3:PutBucketObjectLockConfiguration`, `cloudtrail:StopLogging`, `logs:Delete*`.

### 2.4. CloudWatch Logs Insights (nếu chưa đủ)

AUDIT-014 cũng dùng Insights trên log group đã có. Nếu profile chưa có:

| Quyền | Mục đích |
|---|---|
| `logs:StartQuery` | Chạy Insights |
| `logs:GetQueryResults` | Đọc kết quả |
| `logs:StopQuery` | Hủy query |
| `logs:DescribeLogGroups` / `logs:DescribeLogStreams` | Chọn đúng log group |
| `logs:FilterLogEvents` | Fallback CLI (thường đã có từ Mandate-04) |

Log groups:

- `/aws/eks/techx-tf4-cluster/cluster`
- `/aws/cloudtrail/tf4-general-cloudtrail`

---

## 3. Policy statement đề xuất

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AuditAthenaQueryRead",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults",
        "athena:StopQueryExecution",
        "athena:ListWorkGroups",
        "athena:GetWorkGroup",
        "athena:ListQueryExecutions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AuditGlueCatalogRead",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:GetTable",
        "glue:GetTables",
        "glue:GetPartition",
        "glue:GetPartitions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AuditLogBucketsAthenaRead",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::tf4-cloudtrail-logs-bucket-*",
        "arn:aws:s3:::tf4-cloudtrail-logs-bucket-*/*",
        "arn:aws:s3:::tf4-eks-audit-logs-*",
        "arn:aws:s3:::tf4-eks-audit-logs-*/*"
      ]
    },
    {
      "Sid": "AthenaQueryResultsReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload"
      ],
      "Resource": [
        "arn:aws:s3:::tf4-athena-query-results-*",
        "arn:aws:s3:::tf4-athena-query-results-*/cdo07/*"
      ]
    },
    {
      "Sid": "AuditLogsInsightsQuery",
      "Effect": "Allow",
      "Action": [
        "logs:StartQuery",
        "logs:GetQueryResults",
        "logs:StopQuery",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:FilterLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

> CDO08 có thể thu hẹp `Resource` Athena/Glue theo ARN workgroup/database cụ thể sau khi CDO04 tạo resource. Statement trên là baseline để PoC không bị chặn.

---

## 4. Ranh giới trách nhiệm

- **CDO08**: cập nhật Permission Set `TF4-AuditReadOnlyAndAnalyze`; đảm bảo không mở quyền delete trên audit SoT buckets.
- **CDO04**: tạo Athena workgroup / Glue table / result bucket (nếu dùng Terraform) trước hoặc song song PoC.
- **CDO07**: sau khi có quyền — chạy PoC AUDIT-014, xác nhận hết `AccessDenied`, lưu evidence, đo cost.

---

## 5. Tiêu chí nghiệm thu (Acceptance Criteria / Evidence)

- [ ] Permission set đã cập nhật trên SSO.
- [ ] CDO07 chạy được `athena:StartQueryExecution` + `GetQueryResults` trên table CloudTrail (hoặc query thông báo rõ nếu table chưa tạo — khi đó không còn AccessDenied IAM).
- [ ] CDO07 `glue:GetDatabase` / `GetTable` (sau khi catalog tồn tại) không AccessDenied.
- [ ] CDO07 Insights `logs:StartQuery` trên 2 log group forensic không AccessDenied.
- [ ] Xác nhận profile **không** có `s3:DeleteObject` trên `tf4-cloudtrail-logs-bucket-*` / `tf4-eks-audit-logs-*`.
- [ ] CDO07 ghi evidence CLI (pass/fail) vào thư mục evidence của AUDIT-014.

---

## 6. Lệnh nghiệm thu mẫu (sau khi CDO08 xong)

```bash
aws sts get-caller-identity --profile TF4-AuditReadOnlyAndAnalyze-511825856493

# Glue (sau khi DB tồn tại)
aws glue get-database --name tf4_audit --profile TF4-AuditReadOnlyAndAnalyze-511825856493

# Athena workgroup
aws athena get-work-group --work-group tf4-cdo07-audit \
  --profile TF4-AuditReadOnlyAndAnalyze-511825856493

# Insights
aws logs start-query \
  --log-group-name /aws/cloudtrail/tf4-general-cloudtrail \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date -u +%s) \
  --query-string 'fields @timestamp, @message | limit 5' \
  --profile TF4-AuditReadOnlyAndAnalyze-511825856493
```

---

## 7. Related

- Parent / PoC: [AUDIT-014-enable-athena-insights-log-analysis.md](./AUDIT-014-enable-athena-insights-log-analysis.md)
- IAM forensic trước: `AUDIT-010-request-mandate04-forensic-audit-permissions.md`
- Firehose / Object Lock: `AUDIT-013-request-firehose-and-s3-lock-audit-permissions.md`

*(Sau khi hoàn thành, vui lòng tag CDO07 để nghiệm thu và mở khóa PoC AUDIT-014.)*
