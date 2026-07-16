# Phương án đề xuất: Lớp phân tích log phục vụ audit (CDO07)

| | |
|---|---|
| **Nhóm** | CDO07 — Auditability |
| **Ngày** | 16/07/2026 |
| **Trạng thái** | **ĐỀ XUẤT** — sẵn sàng trình bày / đưa backlog |
| **Nguồn kỹ thuật** | Repo `tf4-phase3-repo` · evidence `docs/evidence/mandate-04-forensic/` |
| **Phạm vi** | Sau Mandate-04 (đã ghi + khóa log) — giải pain **đọc / phân tích** |

---

## 1. Tóm tắt điều hành (30 giây)

Mandate-04 đã có đường ghi vết bất biến trên **S3 Object Lock**. Việc còn lại là audit đang phải **tải log từ S3 + CLI/jq**, log rải nhiều chỗ, drill sát 10 phút.

**Phương án đề xuất:**

> **Giữ S3 Object Lock làm nguồn sự thật.**  
> Thêm lớp đọc **pay-per-query**: **Amazon Athena** (đọc thẳng S3) + **CloudWatch Logs Insights** (sự kiện gần trên log group đã có).  
> **Không** triển khai ELK / Datadog / Loki làm nền forensic; **không** ingest audit vào OpenSearch ở giai đoạn này.

**Vì sao đây là phương án tốt nhất:** đúng pain task, rẻ nhất với tần suất audit thưa (~vài cent đến <$1/tuần lớp phân tích), không phá tamper-evident, tái dụng hạ tầng đã có, gắn được backlog Task 3.1/3.2 sẵn có trong team plan.

---

## 2. Bối cảnh & bài toán

### 2.1. Mandate-04 đã xong gì

| Yêu cầu Mandate | Trạng thái (evidence 15–16/07) |
|---|---|
| Ghi vết CloudTrail + EKS audit | PASS — trail + control plane logging |
| Tamper-evident | PASS — Object Lock COMPLIANCE 90 ngày, validate-logs 34/34, separation of duties |
| Forensic ≤10 phút | PASS 3/3 drill — nhưng **sát hạn**, chủ yếu CLI + jq |
| Truy về người | PASS — SSO session → danh tính |

Pipeline hiện tại (~170 MB/ngày): **~$0.64/tuần** — giữ nguyên, không thay.

### 2.2. Pain còn lại (= task này)

1. Log nằm nhiều chỗ: CloudWatch (EKS + CloudTrail), CloudTrail API, 2 bucket S3, AWS Config, app log OpenSearch.
2. Audit sâu phải **download S3** rồi chạy code/script.
3. Không có lớp query thân thiện / saved query chuẩn; khó người mới và khó lặp lại.

→ Cần **lớp phân tích**, không làm lại lớp ghi/khóa.

---

## 3. Phương án được đề xuất

### 3.1. Tên phương án

**“Athena + Insights trên SoT S3”**  
(Dual-read, single source of truth)

### 3.2. Kiến trúc

```
┌──────────────────────────────────────────────────────────┐
│  SOURCE OF TRUTH (đã có — KHÔNG thay thế)                 │
│  S3 Object Lock COMPLIANCE 90 ngày                        │
│  • tf4-cloudtrail-logs-bucket-511825856493                │
│    prefix: AWSLogs/511825856493/CloudTrail/               │
│  • tf4-eks-audit-logs-511825856493  (qua Firehose)        │
└────────────────────────────┬─────────────────────────────┘
                             │ READ ONLY
              ┌──────────────┴──────────────┐
              ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────────┐
│ LỚP A — Amazon Athena   │   │ LỚP B — CloudWatch Insights │
│ (+ Glue catalog)        │   │ (log group đã có)           │
│                         │   │                             │
│ • Query SQL trên S3     │   │ • /aws/eks/.../cluster      │
│ • Không cần download    │   │ • /aws/cloudtrail/          │
│ • Dùng khi audit sâu /  │   │   tf4-general-cloudtrail    │
│   ngoài cửa sổ CWL      │   │ • Dùng khi drill gần        │
└────────────┬────────────┘   └──────────────┬──────────────┘
             │                               │
             └───────────────┬───────────────┘
                             ▼
              Auditor / Mentor demo / Grafana Explore
              (mở rộng Task 3.2 Grafana Audit Dashboard)
```

### 3.3. Quy tắc sử dụng (để team không loạn)

| Tình huống | Dùng gì |
|---|---|
| Mentor hỏi sự kiện trong vài giờ / vài ngày gần | **Logs Insights** (EKS hoặc CloudTrail CWL) |
| Cần scan S3, ngoài retention CWL, hoặc chứng minh không download | **Athena** |
| Mentor hỏi “log có sửa được không?” | **S3 Object Lock + validate-logs** (không dùng index) |
| App error / checkout log | OpenSearch `otel-logs-*` (CDO08) — **không** trộn với audit trail AWS/K8s |

### 3.4. Thành phần triển khai tối thiểu

| Thành phần | Giá trị |
|---|---|
| Region | `us-east-1` |
| Athena workgroup | `tf4-cdo07-audit` |
| Glue database | `tf4_audit` |
| Tables | `cloudtrail_events`, `eks_audit_events` (PoC có thể làm CT trước) |
| Result location | `s3://tf4-athena-query-results-<account>/cdo07/` — lifecycle 7–30 ngày, **không** Object Lock |
| IAM | Bổ sung read-only Athena/Glue/GetObject cho `TF4-AuditReadOnlyAndAnalyze` |
| Insights | Saved queries trên 2 log group ở trên (zero infra mới) |

---

## 4. Vì sao đây là phương án tốt nhất

### 4.1. So với các lựa chọn thị trường trong task

| Giải pháp | Quyết định | Lý do |
|---|---|---|
| **Athena (+ Insights)** | **CHỌN** | Đúng pain download S3; pay-per-query; giữ Object Lock; fit volume nhỏ + audit thưa |
| **OpenSearch** (đã có) | **Hoãn (P2)** | Security plugin tắt; PVC 8Gi đã watermark; ingest audit = nhân đôi lưu + phí cố định |
| **ELK** | **Loại** | Trùng năng lực OpenSearch; thêm stack = phí vận hành |
| **Loki** | **Hoãn** | Rẻ cho app log; kém fit JSON CloudTrail/EKS lớn trừ khi parse nặng |
| **Datadog** | **Loại** | Cost ingest/host; data rời account; không thay SoT S3 |

### 4.2. Tiêu chí quyết định (đã cân)

| Tiêu chí | Athena + Insights | OpenSearch ingest ngay | Datadog | ELK |
|---|---|---|---|---|
| Giải “phải down S3” | ✅ | ✅ (sau ingest) | ✅ | ✅ |
| Chi phí với audit thưa | ✅ Rẻ nhất | ❌ Phí cố định | ❌ | ❌ |
| Giữ tamper-evident SoT | ✅ Chỉ đọc S3 | ⚠️ Index xóa được | ⚠️ | ⚠️ |
| Fit stack TF4 hiện tại | ✅ | ⚠️ Đang đầy đĩa | ❌ | ❌ |
| Thời gian có PoC | ✅ Nhanh (IAM + Glue) | ❌ Chờ CDO08 harden | Trung bình | Chậm |
| Demo mentor / drill ≤10 phút | ✅ Insights gần + Athena sâu | UI tốt nhưng blocker | UI tốt | UI tốt |

### 4.3. Cost (lý do “tối ưu ngân sách”)

- Pipeline ghi hiện tại: **~$0.64/tuần** — giữ.
- Athena: **$5/TB scanned** → với partition theo ngày + ~20 query/tuần thường **cents–<$1/tuần**.
- Insights: **$0.005/GB scanned** → hẹp time window → rất thấp.
- **Ceiling đề xuất:** lớp phân tích **≤ $5/tuần** (kỳ vọng thực tế **<$1/tuần**).

**Không làm để tiết kiệm:** ingest audit vào OpenSearch; Datadog/ELK; kéo dài retention CloudWatch chỉ để query dễ; scheduled query rộng.

---

## 5. Việc không làm (phạm vi loại trừ)

1. Không tắt / thay Firehose, Object Lock, CloudTrail validation.
2. Không coi OpenSearch/Grafana là bằng chứng bất biến trước mentor.
3. Không dựng ELK riêng hoặc Datadog làm forensic SoT trong phase này.
4. Không trộn index `audit-*` với `otel-logs-*` nếu sau này làm P2.

---

## 6. Lộ trình & backlog

### 6.1. Thứ tự triển khai

| Giai đoạn | Việc | Owner gợi ý | DoD ngắn |
|---|---|---|---|
| **Ngay** | Chốt phương án này trong họp nhóm | CDO07 | Đồng ý ADR |
| **P0** | PoC Athena trên CloudTrail S3 | CDO07 + CDO04 (IAM) | ≥1–3 query map drill; không `s3 cp`; đo thời gian vs CLI |
| **P1** | Saved Logs Insights (EKS + CloudTrail CWL) | CDO07 | ≥3 saved query; cập nhật playbook |
| **P1** | Extend Task 3.2 Grafana Audit Dash | Member 8 / CDO07 | Link Explore / panel forensic (401/403 + query mẫu) |
| **Docs** | Sửa playbook còn ghi `LogFileValidationEnabled=false` | CDO07 | Khớp evidence AUD-17.1/17.3 |
| **P2 (sau)** | OpenSearch `audit-*` | CDO08 + Member 7 (ISM) | Chỉ khi PVC/capacity OK + security plugin + ISM |

### 6.2. Gắn kế hoạch sẵn trong repo

- `docs/audit/DELEGATED_TASKS_P0.md` — Task 3.1 ISM, Task 3.2 Grafana Audit Dashboard  
- `docs/audit/TEAM_ASSIGNMENT.md` — Member 7 / Member 8  
- `docs/audit/JIRA_TASKS.md` — Task 7–8  

### 6.3. Definition of Done cấp phương án (khi nào coi “xong bài”)

- [ ] Auditor hoàn thành ≥1 scenario kiểu AUD-17.2 **không** dùng `aws s3 cp`.
- [ ] Có saved Insights và/hoặc Athena query tái sử dụng được.
- [ ] Chi phí lớp phân tích trong ceiling ≤ $5/tuần (đo Cost Explorer).
- [ ] Runbook forensic cập nhật (CLI = fallback).
- [ ] Tài liệu khẳng định: **S3 Object Lock vẫn là SoT**.

---

## 7. Rủi ro & giảm thiểu

| Rủi ro | Giảm thiểu |
|---|---|
| Athena scan cả bucket → bill bất ngờ | Workgroup limit bytes/query; bắt buộc filter ngày / partition |
| Schema Glue sai | Dùng template CloudTrail Athena chuẩn; validate bằng event đã biết từ drill |
| Mentor bắt demo CLI raw | Giữ playbook CLI; Athena/Insights là đường “dễ hơn” |
| Nhầm OpenSearch = SoT | Nêu rõ trong demo: SoT = S3 Object Lock |
| Thiếu quyền Athena/Glue | Ticket IAM bổ sung read-only (kiểu AUDIT-010) |

---

## 8. Query / PoC mẫu (để demo ý tưởng)

### 8.1. Athena — StartSession (on-call)

```sql
SELECT
  from_iso8601_timestamp(eventtime) AS event_time,
  eventname,
  useridentity.arn AS principal_arn,
  sourceipaddress,
  useragent
FROM tf4_audit.cloudtrail_events
WHERE eventname = 'StartSession'
  AND from_iso8601_timestamp(eventtime)
      >= current_timestamp - INTERVAL '7' DAY
ORDER BY event_time DESC
LIMIT 50;
```

### 8.2. Logs Insights — hướng EKS (chỉnh field sau PoC)

```
fields @timestamp, @message
| filter @message like /portforward/
| sort @timestamp desc
| limit 50
```

Log groups:

- `/aws/eks/techx-tf4-cluster/cluster`
- `/aws/cloudtrail/tf4-general-cloudtrail`

---

## 9. Script trình bày ~2 phút

> Mandate-04 đã xong ghi và khóa log trên S3 Object Lock; drill forensic pass nhưng còn phải CLI và đôi khi download S3.  
> Chúng tôi so sánh OpenSearch, ELK, Loki, Datadog. OpenSearch đang tắt security và đầy đĩa — không ingest audit lúc này. ELK trùng stack. Datadog đắt. Loki kém fit JSON audit.  
> **Chốt:** Athena đọc thẳng S3 giải pain download; Insights trên CloudWatch đã có cho drill gần; cost kỳ vọng dưới 1 đô/tuần; S3 vẫn là nguồn sự thật.  
> Xin nhóm duyệt để mở PoC Athena + saved Insights và gắn vào Task 3.2 Grafana sẵn có.

---

## 10. Quyết định cần lấy từ họp

| # | Câu hỏi | Đề xuất sẵn |
|---|---|---|
| 1 | Duyệt phương án Athena + Insights? | **Có** |
| 2 | Ceiling cost lớp phân tích | **≤ $5/tuần** |
| 3 | Có mở OpenSearch ingest audit ngay? | **Không** — P2 sau capacity + security |
| 4 | Ai xin IAM Athena/Glue? | CDO07 tạo ticket → CDO04/IAM |
| 5 | Có sửa playbook drift validation? | **Có** (P0 docs) |

---

## Phụ lục — Tham chiếu nhanh

| Mục | Giá trị / đường dẫn trong repo |
|---|---|
| CloudTrail | `tf4-general-cloudtrail` |
| CT S3 | `tf4-cloudtrail-logs-bucket-511825856493` |
| CT CWL | `/aws/cloudtrail/tf4-general-cloudtrail` |
| EKS CWL | `/aws/eks/techx-tf4-cluster/cluster` |
| EKS S3 | `tf4-eks-audit-logs-511825856493` |
| Firehose | `tf4-eks-audit-logs-firehose` |
| Profile audit | `TF4-AuditReadOnlyAndAnalyze` |
| Evidence | `docs/evidence/mandate-04-forensic/` |
| Framework | `docs/audit/tickets/AUDIT-CDO07-MANDATE04-FORENSIC-FRAMEWORK.md` |
| Playbook | `docs/audit/runbooks/forensic-playbook-timeline.md` |

---

**Người soạn:** Trieu Nguyen - CDO07 (research task phân tích giải pháp log)  
**Khuyến nghị hành động tiếp theo:** duyệt §10 → tạo ticket IAM + PoC Athena trong sprint tới.
