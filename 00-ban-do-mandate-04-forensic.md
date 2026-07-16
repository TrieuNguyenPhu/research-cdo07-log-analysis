# Bản đồ toàn bộ Mandate-04 Forensic

> **Tài liệu research nằm ngoài repo:** `C:\Users\NguyenPhuTrieu\Downloads\research-cdo07-log-analysis`  
> Nguồn chính **trong** repo `tf4-phase3-repo` (đọc khi cần evidence thô):
> - Mandate: `mandates/MANDATE-04-auditability-tf4.md`
> - Framework: `docs/audit/tickets/AUDIT-CDO07-MANDATE04-FORENSIC-FRAMEWORK.md`
> - Evidence: `docs/evidence/mandate-04-forensic/*`
> - Runbook: `docs/audit/runbooks/forensic-playbook-timeline.md`
> - Jira mapping: `docs/audit/JIRA_TASKS.md` (Task 19–23)

---

## 1. Mandate muốn chứng minh điều gì?

Ban kiểm toán **không** hỏi “hệ thống có chạy không”. Họ hỏi:

> Chuyện gì đã xảy ra? Ai làm? Khi nào? Và làm sao tin được bản ghi đó?

| # | Yêu cầu Mandate | Nghĩa thực tế với CDO07 |
|---|---|---|
| 1 | Đường ghi vết đầy đủ | CloudTrail (AWS API) + EKS audit (K8s) + change trail (AWS Config / infra) |
| 2 | Bài forensic chấm tại chỗ | Mentor chọn 1 sự kiện → dựng WHO/WHAT/WHEN/HOW **≤10 phút**, chỉ từ log |
| 3 | Tamper-evident | Operator không tự xóa/sửa vết của mình; có bằng chứng integrity |
| 4 | Truy về người | Không shared account; ARN/session → danh tính cá nhân |

Ràng buộc: **~$300/tuần/TF**, không đụng flagd, storefront vẫn public / ops vẫn private.

---

## 2. Cấu trúc công việc CDO07 (4 subtask AUD-17.x)

```
MANDATE-04 Forensic
├── AUD-17.1  Verify coverage          → CloudTrail + EKS audit + AWS Config có ghi đủ
├── AUD-17.2  Forensic drill           → ≥5 scenario thiết kế, ≥3 drill ≤10 phút
├── AUD-17.3  Tamper-evident           → Object Lock + separation of duties + validate-logs
└── AUD-17.4  Identity mapping         → ≥5 hành động → người thật; không shared account
```

Prerequisite đã/đang làm qua ticket delegated:

| Ticket | Việc | Ai làm |
|---|---|---|
| AUDIT-010 | Cấp quyền read-only forensic (`TF4-AuditReadOnlyAndAnalyze`) | IAM / CDO liên quan |
| AUDIT-011 | Fix CloudTrail Terraform (log validation, multi-region…) | CDO04 Platform |
| AUDIT-013 (+ related) | Firehose / S3 lock / quyền đọc evidence | Platform + CDO07 |

---

## 3. Kiến trúc đã dựng (lớp “ghi + khóa”)

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│  AWS API calls              │     │  EKS Control Plane           │
│  (console / CLI / SSO)      │     │  api, audit, authenticator…  │
└─────────────┬───────────────┘     └──────────────┬───────────────┘
              │                                    │
              ▼                                    ▼
     CloudTrail tf4-general-cloudtrail    CloudWatch Logs
     (multi-region,                         /aws/eks/.../cluster
      LogFileValidation ON)                 retention 7 ngày
              │                                    │
              │                          Subscription Filter
              │                                    ▼
              │                         Firehose tf4-eks-audit-logs-firehose
              │                                    │
              ▼                                    ▼
   S3 CloudTrail logs bucket              S3 EKS audit logs bucket
   Object Lock COMPLIANCE 90d             Object Lock COMPLIANCE 90d
   + Versioning + SSE                     + Versioning + SSE
```

**Ý tưởng cốt lõi:** CloudWatch chỉ là “cửa sổ gần” (7 ngày). Bản bất biến dài hạn nằm trên **S3 Object Lock COMPLIANCE** — kể cả root cũng không xóa trước hạn retention.

Chi phí luồng này (~170 MB/ngày): **~$0.64/tuần** — rất nhỏ so với ngân sách.

---

## 4. Bản đồ thư mục evidence `docs/evidence/mandate-04-forensic/`

### 4.1. AUD-17.1 — Coverage (log có ghi không?)

| File | Nội dung |
|---|---|
| `aud-17.1-cloudtrail-config.md` | Cấu hình CloudTrail |
| `aud-17.1-cloudtrail-status.md` | IsLogging / multi-region |
| `aud-17.1-aws-config-status.md` | AWS Config recorder |
| `aud-17.1-eks-audit-config.md` | EKS control plane logging |
| `aud-17.1-query-test-result.md` | Query thử CloudTrail + EKS có event |

### 4.2. AUD-17.2 — Forensic drill (dựng timeline được không?)

| File | Nội dung |
|---|---|
| `aud-17.2-scenario-01-infra-change.md` | Scenario: thay đổi hạ tầng |
| `aud-17.2-scenario-02-k8s-cluster-access.md` | Scenario: truy cập / thao tác K8s |
| `aud-17.2-scenario-03-unauthorized-access.md` | Scenario: AccessDenied |
| `aud-17.2-scenario-04-secrets-access.md` | Scenario: secrets / KMS / SSM |
| `aud-17.2-scenario-05-oncall-action.md` | Scenario: on-call (SSM StartSession…) |
| `aud-17.2-drill-log.md` | **Kết quả drill thật 3/3 PASS** (stopwatch) |

Drill thật (15/07/2026): ~6m50s / ~6–9m / ~6m20s — **pass nhưng sát 10 phút**, chủ yếu vì CLI + epoch + jq.

### 4.3. AUD-17.3 — Tamper-evident (log có bị sửa/xóa được không?)

| File | Nội dung |
|---|---|
| `aud-17.3-separation-test.md` | Operator thử xóa/stop → AccessDenied |
| `aud-17.3-s3-object-lock-test.md` | Object Lock COMPLIANCE |
| `aud-17.3-validate-logs-test.md` | CloudTrail digest SHA-256 |
| `aud-17.3-firehose-config.md` | Firehose stream EKS → S3 |
| `Báo cáo temper-evident.md` / `Báo cáo check s3 bucket.md` | Báo cáo tổng hợp |

### 4.4. AUD-17.4 — Identity mapping (truy về người)

| File | Nội dung |
|---|---|
| `aud-17.4-identity-mapping.md` | Bảng người ↔ SSO ↔ K8s group |
| `aud-17.4-iam-users-inventory.json` | Inventory IAM users |
| `aud-17.4-infra-changes-7days.json` | Sample change trail |
| `aud-17.4-bastion-access-7days.json` | Sample on-call / bastion |

CDO07 dùng permission set `TF4-AuditReadOnlyAndAnalyze` + K8s group `audit-readonly-analyzers`.

### 4.5. Báo cáo tổng hợp

| File | Nội dung |
|---|---|
| `Báo cáo nghiên cứu Mandate-4.md` | Nghiên cứu Object Lock, Firehose, chi phí, timeline vá S3 |

---

## 5. Quy trình forensic hiện tại (playbook)

Khi mentor đưa sự kiện:

1. **Phân loại** — trong cluster → K8s audit (CloudWatch); AWS API → CloudTrail.
2. **Tính epoch ms** — CloudWatch cần start/end theo milliseconds.
3. **`aws logs filter-log-events` / `aws cloudtrail lookup-events`** — CLI.
4. **`jq` parse JSON** — lọc verb/user/resource/errorCode.
5. **Trình bày WHO / WHAT / WHEN / HOW**.

Tài liệu vận hành: `docs/audit/runbooks/forensic-playbook-timeline.md`.

---

## 6. Phân biệt: Mandate-04 đã xong vs task mới của bạn

| Lớp | Trạng thái (theo evidence 15/07) | Task mới? |
|---|---|---|
| **Ghi log** (CloudTrail, EKS audit, Firehose) | Đã có | Không — đã nghiệm thu hạ tầng |
| **Khóa log** (Object Lock, Deny operator, validate-logs) | Đã có + test | Không |
| **Map danh tính** (SSO session → người) | Đã có bảng mapping | Không (duy trì) |
| **Đọc / phân tích / UI / query tập trung** | **Yếu** — CLI + jq + download S3 | **Đúng task của bạn** |

### Pain đúng như mô tả task

1. **Log lung tung:** CloudWatch (EKS 7 ngày) / CloudTrail API / S3 CloudTrail / S3 EKS / (app log ở OpenSearch của CDO08 — nguồn khác nữa).
2. **Audit sâu phải down S3:** file `.json.gz` partition theo giờ → `aws s3 cp` + script local.
3. **Không có lớp search thân thiện** cho audit trail bất biến → khó demo mentor, khó người mới, sát 10 phút.

→ Cần giải pháp **phân tích log** (search/index/UI) đặt **lên trên** S3 bất biến, **không thay** Object Lock.

---

## 7. Liên hệ với observability đã có trong cluster (CDO08)

Trong namespace `techx-observability` đã có:

- **Grafana** (+ Explore)
- **OpenSearch** (index kiểu `otel-logs-*` — chủ yếu **app/OTel logs**, Jaeger traces)
- **Prometheus / Jaeger**

Lưu ý quan trọng khi nghiên cứu giải pháp:

- OpenSearch hiện tại **không phải** kho audit trail Mandate-04 (CloudTrail / EKS audit bất biến).
- Scan CDO07 từng ghi: OpenSearch security plugin disable, persistence từng là concern — nếu tái dùng OpenSearch cho forensic phải xử lý bảo mật + retention riêng.
- App log (checkout/payment) và audit log (CloudTrail/EKS) là **hai bài toán khác nhau**; có thể dùng chung nền tảng search nhưng **không trộn role**: audit store vẫn là S3 Object Lock.

---

## 8. Checklist “tôi đã hiểu Mandate-04 chưa?”

Bạn hiểu đủ khi trả lời được:

1. Mentor hỏi “ai port-forward Grafana lúc incident?” — mình query **nguồn nào**, lệnh nào?
2. Vì sao CloudWatch 7 ngày vẫn “đủ” cho drill ngắn hạn nhưng **không đủ** làm source of truth kiểm toán?
3. Object Lock COMPLIANCE khác GOVERNANCE chỗ nào (với bài “root cũng không xóa được”)?
4. Firehose giải quyết lỗ hổng nào của CloudWatch?
5. Task phân tích log giải quyết pain **đọc**, không phải pain **ghi/khóa** đã xong?

Nếu 5 câu trên rõ → đủ nền để chọn OpenSearch / ELK / Loki / Datadog / Athena đưa backlog.
