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
   │ A. Athena     │     │ B. CloudWatch   │     │ C. OpenSearch    │
   │ + Glue table  │     │ Logs Insights   │     │ index audit-*    │
   │ (query S3)    │     │ (EKS ≤7 ngày)   │     │ (P2, optional)   │
   └───────┬───────┘     └────────┬────────┘     └────────┬─────────┘
           │                      │                       │
           └──────────────────────┼───────────────────────┘
                                  ▼
                    Auditor / Mentor demo
                    (Athena console hoặc Grafana Explore)
```

| Lớp | Vai trò | Khi dùng |
|---|---|---|
| **S3 Object Lock** | Bản ghi bất biến, chứng minh tamper-evident | Luôn luôn — SoT |
| **A. Athena** | Audit sâu, >7 ngày, không download file | Mọi lần “phải down S3” trước đây |
| **B. CloudWatch Insights / Grafana** | Drill nhanh sự kiện gần (EKS 7 ngày) | Mentor hỏi sự kiện trong tuần |
| **C. OpenSearch `audit-*`** | UI search full-text sau khi sec plugin OK | Chỉ khi CDO08 harden xong |

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

## 4. Phương án B — CloudWatch Logs Insights / Grafana (làm song song nhẹ)

Không cần ingest mới. EKS audit **đã** ở `/aws/eks/techx-tf4-cluster/cluster`.

**Mục tiêu:** thay `filter-log-events` + jq bằng query Insights (hoặc Grafana Explore → CloudWatch).

Ví dụ Logs Insights:

```
fields @timestamp, user.username, verb, objectRef.resource, objectRef.name, responseStatus.code
| filter objectRef.resource = "pods/portforward" or ispresent(objectRef.resource)
| filter @message like /portforward/
| sort @timestamp desc
| limit 50
```

(Điều chỉnh field path theo JSON audit thật sau khi parse `@message`.)

**Ưu:** Zero infra mới, giảm thời gian epoch/jq trong drill 7 ngày.  
**Nhược:** Không giải pain “>7 ngày phải down S3” → vẫn cần Athena.

---

## 5. Phương án C — OpenSearch (chỉ sau khi CDO08 harden)

Chỉ mở khi:

1. OpenSearch **security plugin bật** (hiện `DISABLE_SECURITY_PLUGIN=true` là blocker).
2. Index riêng `audit-*` / `cloudtrail-*` — **không** trộn `otel-logs-*`.
3. ISM retention rõ; runbook ghi: lệch index ↔ S3 thì **tin S3 + Athena**.

Pipeline gợi ý: CloudWatch subscription hoặc định kỳ sync từ S3 → OpenSearch; Grafana Explore làm UI.

**Không** coi OpenSearch là bằng chứng tamper-evident trước mentor.

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
| W1 | Chốt ADR (file `05`) + trình bày nhóm | CDO07 | Decision signed |
| W1–W2 | Ticket IAM Athena/Glue read-only | CDO07 → CDO04/IAM | Permission set cập nhật |
| W2 | PoC Athena 2–3 query map drill AUD-17.2 | CDO07 | Bảng thời gian CLI vs Athena |
| W2 | Saved CloudWatch Insights / Grafana (EKS 7d) | CDO07 + CDO08 | 2 query mẫu trong runbook draft |
| Sau | OpenSearch ingest audit (nếu ADR cho phép) | CDO08 lead + CDO07 | Index + ACL |

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
