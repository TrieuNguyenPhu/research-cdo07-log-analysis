# Phương án đề xuất: Lớp phân tích log phục vụ audit

| | |
|---|---|
| **Nhóm** | CDO07 — Auditability |
| **Người soạn** | Trieu Nguyen |
| **Ngày** | 16/07/2026 |
| **Trạng thái** | ĐỀ XUẤT — xin nhóm duyệt |
| **Task** | Phân tích các giải pháp phân tích log hiện tại → đưa backlog |
| **Nguồn** | Repo `tf4-phase3-repo` · `docs/evidence/mandate-04-forensic/` |

---

## 1. Tóm tắt điều hành (30 giây)

Mandate-04 đã **ghi và khóa** audit log trên S3 Object Lock. Pain còn lại: log rải nhiều chỗ, audit sâu phải **download S3 + CLI/jq**, drill sát 10 phút.

**Đề xuất:**

> Giữ **S3 Object Lock** làm nguồn sự thật.  
> Thêm **Amazon Athena** (CloudTrail + EKS [+ Config]) đọc thẳng S3 + **CloudWatch Logs Insights** cho drill gần.  
> Không ELK / Datadog; không ingest OpenSearch lúc này; Athena **không** phải “real-time stream”.

Chi tiết DDL/query/cost: [05-architecture-athena-forensics.md](./05-architecture-athena-forensics.md).

**Vì sao:** đúng pain, rẻ với audit thưa, không phá tamper-evident, tái dụng hạ tầng, gắn Task 3.1/3.2.

---

## 2. Bối cảnh

### 2.1. Mandate-04 đã xong

| Yêu cầu | Trạng thái (evidence 15–16/07) |
|---|---|
| CloudTrail + EKS audit | PASS |
| Tamper-evident (Object Lock 90d, validate-logs, SoD) | PASS |
| Forensic ≤10 phút | PASS 3/3 — sát hạn, CLI + jq |
| Truy về người (SSO) | PASS |

Pipeline ghi (~170 MB/ngày): **~$0.64/tuần** — **giữ nguyên**.

### 2.2. Pain = scope task này

1. Log nhiều chỗ: CWL EKS, CWL CloudTrail, API CloudTrail, 2 bucket S3, Config, OpenSearch app.
2. Audit sâu phải `aws s3 cp` + script.
3. Chưa có saved query / quy trình đọc chuẩn.

→ Cần **lớp phân tích**, không làm lại lớp ghi/khóa.

---

## 3. Phương án đề xuất: “Athena + Insights trên SoT S3”

### 3.1. Kiến trúc

```
┌──────────────────────────────────────────────────────────┐
│  SOURCE OF TRUTH (đã có — không thay)                     │
│  S3 Object Lock COMPLIANCE 90 ngày                        │
│  • tf4-cloudtrail-logs-bucket-511825856493                │
│  • tf4-eks-audit-logs-511825856493                        │
└────────────────────────────┬─────────────────────────────┘
                             │ READ ONLY
              ┌──────────────┴──────────────┐
              ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────────┐
│ A. Amazon Athena        │   │ B. CloudWatch Logs Insights │
│ (+ Glue)                │   │ (log group đã có)           │
│ Query SQL trên S3       │   │ /aws/eks/.../cluster        │
│ Không download          │   │ /aws/cloudtrail/            │
│ Audit sâu / ngoài CWL   │   │   tf4-general-cloudtrail    │
└────────────┬────────────┘   └──────────────┬──────────────┘
             └───────────────┬───────────────┘
                             ▼
              Auditor / Mentor / Grafana Explore
              (mở rộng Task 3.2 Grafana Audit Dashboard)
```

### 3.2. Khi nào dùng gì

| Tình huống | Dùng |
|---|---|
| Sự kiện vài giờ / vài ngày gần | **Logs Insights** |
| Scan S3, ngoài cửa sổ CWL, chứng minh không download | **Athena** |
| “Log có sửa được không?” | **S3 Object Lock + validate-logs** |
| App / checkout log | OpenSearch `otel-logs-*` (CDO08) — **không** trộn audit AWS/K8s |

### 3.3. Thành phần PoC tối thiểu

| Thành phần | Giá trị |
|---|---|
| Region | `us-east-1` |
| Athena workgroup | `tf4-cdo07-audit` (limit bytes scanned) |
| Glue DB / tables | `tf4_audit` · `cloudtrail_events` (+ EKS sau) |
| Result S3 | `s3://tf4-athena-query-results-<account>/cdo07/` · lifecycle 7–30 ngày |
| IAM | Athena/Glue/S3 GetObject read-only cho `TF4-AuditReadOnlyAndAnalyze` |
| Insights | Saved query trên 2 log group ở trên |

---

## 4. So sánh nhanh & vì sao chọn

| Giải pháp | Quyết định | Lý do ngắn |
|---|---|---|
| **Athena + Insights** | **Chọn** | Đúng pain download; pay-per-query; giữ Object Lock |
| OpenSearch (đã có) | Hoãn P2 | Security tắt; PVC 8Gi watermark; phí cố định nếu ingest |
| ELK | Loại | Trùng OpenSearch |
| Loki | Hoãn | Kém fit JSON audit lớn |
| Datadog | Loại | Cost + data ra ngoài account |

Chi tiết bảng tiêu chí / cost: xem [02-phu-luc-so-sanh.md](./02-phu-luc-so-sanh.md).

### Cost

| Mục | Ước |
|---|---|
| Pipeline ghi hiện tại | ~$0.64/tuần (giữ) |
| Athena ($5/TB) + Insights ($0.005/GB) | Thường **cents – <$1/tuần** nếu partition + hẹp time range |
| **Ceiling đề xuất** | **≤ $5/tuần** cho lớp phân tích |

---

## 5. Phạm vi loại trừ

1. Không tắt/thay Firehose, Object Lock, CloudTrail log validation.
2. Không coi OpenSearch/Grafana là bằng chứng bất biến.
3. Không dựng ELK / Datadog forensic SoT trong phase này.
4. Không ingest audit vào OpenSearch trước khi có capacity + security plugin.

---

## 6. Lộ trình sau khi duyệt

| Giai đoạn | Việc | Owner | DoD ngắn |
|---|---|---|---|
| **Ngay** | Duyệt phương án | CDO07 | Checklist §10 |
| **P0** | PoC Athena (CloudTrail S3 trước) | CDO07 + CDO04 IAM | ≥1–3 query map drill; không `s3 cp` |
| **P1** | Saved Insights EKS + CloudTrail CWL | CDO07 | ≥3 saved query |
| **P1** | Extend Task 3.2 Grafana Audit Dash | Member 8 | Link Explore / panel forensic |
| **Docs** | Sửa playbook (`LogFileValidationEnabled` vẫn ghi false) | CDO07 | Khớp AUD-17.1/17.3 |
| **P2** | OpenSearch `audit-*` (nếu cần) | CDO08 + Member 7 ISM | PVC/sec/ISM OK |

Stories chi tiết: [03-backlog.md](./03-backlog.md).

### DoD cấp phương án

- [ ] ≥1 scenario kiểu AUD-17.2 **không** `aws s3 cp`
- [ ] Có saved Insights và/hoặc Athena query
- [ ] Cost lớp phân tích ≤ $5/tuần
- [ ] Runbook cập nhật (CLI = fallback)
- [ ] S3 Object Lock vẫn là SoT

---

## 7. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Athena scan cả bucket | Workgroup limit bytes; filter theo ngày |
| Schema Glue sai | Template CloudTrail Athena chuẩn + event drill đã biết |
| Mentor bắt CLI | Giữ playbook CLI làm fallback |
| Nhầm OpenSearch = SoT | Nêu rõ trong demo |
| Thiếu IAM | Ticket read-only kiểu AUDIT-010 |

---

## 8. Query mẫu (ý tưởng PoC)

**Athena — StartSession**

```sql
SELECT
  from_iso8601_timestamp(eventtime) AS event_time,
  eventname,
  useridentity.arn AS principal_arn,
  sourceipaddress
FROM tf4_audit.cloudtrail_events
WHERE eventname = 'StartSession'
  AND from_iso8601_timestamp(eventtime)
      >= current_timestamp - INTERVAL '7' DAY
ORDER BY event_time DESC
LIMIT 50;
```

**Insights — EKS (chỉnh field sau PoC)**

```
fields @timestamp, @message
| filter @message like /portforward/
| sort @timestamp desc
| limit 50
```

---

## 9. Script trình bày ~2 phút

> Mandate-04 đã ghi và khóa log trên S3 Object Lock; drill pass nhưng còn CLI và đôi khi phải download S3.  
> So sánh OpenSearch, ELK, Loki, Datadog: OpenSearch đang tắt security và đầy đĩa — chưa ingest audit. ELK trùng. Datadog đắt. Loki kém fit JSON audit.  
> **Đề xuất:** Athena đọc thẳng S3; Insights trên CloudWatch đã có cho drill gần; cost kỳ vọng dưới 1 đô/tuần; S3 vẫn là nguồn sự thật.  
> Xin duyệt để mở PoC Athena + saved Insights và gắn Task 3.2 Grafana.

---

## 10. Xin nhóm quyết định

| # | Câu hỏi | Đề xuất |
|---|---|---|
| 1 | Duyệt Athena + Insights? | **Có** |
| 2 | Ceiling cost lớp phân tích | **≤ $5/tuần** |
| 3 | Ingest OpenSearch audit ngay? | **Không** — P2 |
| 4 | Ai xin IAM Athena/Glue? | CDO07 ticket → CDO04/IAM |
| 5 | Sửa drift playbook validation? | **Có** |

---

## Phụ lục — Tham chiếu

| Mục | Giá trị |
|---|---|
| CloudTrail | `tf4-general-cloudtrail` |
| CT S3 | `tf4-cloudtrail-logs-bucket-511825856493` |
| CT CWL | `/aws/cloudtrail/tf4-general-cloudtrail` |
| EKS CWL | `/aws/eks/techx-tf4-cluster/cluster` |
| EKS S3 | `tf4-eks-audit-logs-511825856493` |
| Firehose | `tf4-eks-audit-logs-firehose` |
| Profile | `TF4-AuditReadOnlyAndAnalyze` |
| Evidence | `docs/evidence/mandate-04-forensic/` |
| Playbook | `docs/audit/runbooks/forensic-playbook-timeline.md` |

**Bước tiếp:** duyệt §10 → CDO08 làm **AUDIT-015** (IAM) → CDO07 làm **AUDIT-014** (PoC).  
Ticket nằm tại: `tickets/AUDIT-014-enable-athena-insights-log-analysis.md`, `tickets/AUDIT-015-request-athena-glue-audit-permissions.md` *(chưa merge repo)*.
