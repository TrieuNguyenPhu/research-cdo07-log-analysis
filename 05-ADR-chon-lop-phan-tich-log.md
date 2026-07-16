# ADR — Chọn lớp phân tích log phục vụ audit (CDO07)

| Trường | Giá trị |
|---|---|
| ADR ID | CDO07-ADR-LOG-001 |
| Trạng thái | **Proposed** (chờ nhóm / mentor chốt) |
| Ngày | 2026-07-16 |
| Người soạn | CDO07 |
| Liên quan | Mandate-04 forensic (đã có SoT S3); task phân tích giải pháp log |

---

## Context

- Audit log đã được ghi và khóa (CloudTrail + EKS → S3 Object Lock COMPLIANCE).
- Quy trình đọc hiện tại: CLI CloudWatch/CloudTrail + jq; audit sâu → download S3 + script.
- Drill forensic pass nhưng sát 10 phút; log rải nhiều chỗ.
- Cần chọn hướng đưa backlog trong các lựa chọn thị trường: OpenSearch, ELK, Loki, Datadog (và AWS-native phù hợp).

## Decision (đề xuất) — cập nhật sau sync repo 16/07/2026

**Chọn kiến trúc 2 lớp đọc + 1 SoT:**

1. **SoT:** S3 Object Lock (không thay).
2. **P0 — Athena (+ Glue)** trên bucket audit (prefix CT: `AWSLogs/511825856493/CloudTrail/`).
3. **P1 — CloudWatch Logs Insights** cho **cả** `/aws/eks/techx-tf4-cluster/cluster` và `/aws/cloudtrail/tf4-general-cloudtrail` (CWL delivery confirmed AUD-17.1); extend Task 3.2 Grafana Audit Dash.
4. **P2 — OpenSearch `audit-*`:** chỉ sau PVC/capacity (incident 8Gi watermark) + security plugin + ISM (Task 3.1).
5. **Loại / hoãn:** ELK; Datadog SoT; Loki-primary CloudTrail; ingest OS ngay.
6. **Docs debt:** sửa playbook vẫn ghi `LogFileValidationEnabled=false`.

Xem nhật ký: `07-sync-repo-2026-07-16.md`.

## Alternatives considered

| Phương án | Kết luận |
|---|---|
| Chỉ OpenSearch ingest ngay | Reject tạm — security plugin đang tắt; risk xóa index |
| ELK song song OpenSearch | Reject — trùng năng lực, tốn vận hành |
| Loki-first | Reject phase này — fit kém JSON audit lớn |
| Datadog | Reject phase này — cost + data egress / SoT |
| Athena-only (không Grafana) | Accept được nếu mentor không cần UI; vẫn khuyến nghị P1 Insights |
| CloudTrail Lake only | Partial — không cover EKS gzip bucket; cân nhắc sau nếu chỉ cần CloudTrail |

## Consequences

**Tích cực**

- Đúng pain task; chi phí PoC thấp; alignment tamper-evident.
- Tái dùng Grafana/CW sẵn có cho drill gần; không bắt buộc dựng stack mới.

**Tiêu cực / nợ kỹ thuật**

- Athena UI kém “sexy” hơn Kibana; cần Glue schema + partition discipline.
- Phụ thuộc ticket IAM (Athena/Glue) từ CDO04.
- OpenSearch ingest trì hoãn → chưa có full-text một chỗ ngay lập tức.

## Cost ceiling (đề xuất ghi vào backlog)

- PoC lớp phân tích: **≤ $5/tuần** (Athena scanned + S3 GET + result bucket).
- Không tăng ingest Datadog/ELK trong phase này.

## Ownership

| Phần | Owner |
|---|---|
| SoT S3 Object Lock / Firehose | Giữ như Mandate-04 (Platform + CDO07 verify) |
| Athena workgroup / Glue tables / saved SQL | **CDO07** (consumer) + CDO04 (IAM/infra nếu cần Terraform) |
| CloudWatch Insights / Grafana datasource | CDO07 dùng; CDO08 hỗ trợ quyền Explore |
| OpenSearch `audit-*` (nếu làm P2) | **CDO08** vận hành; CDO07 định nghĩa query/ACL yêu cầu |

## Validation

ADR được coi là **Accepted** khi:

- [ ] Lead CDO07 confirm
- [ ] CDO04 confirm khả thi IAM/Athena trong budget
- [ ] CDO08 confirm (P1 Grafana OK; P2 OpenSearch có/không roadmap sec plugin)
- [ ] (Optional) Mentor không phản đối Athena làm đường audit chính

## Links

- Giải pháp chi tiết: `04-giai-phap-de-xuat.md`
- So sánh: `02-so-sanh-giai-phap-phan-tich-log.md`
- Backlog stories: `03-de-xuat-backlog.md`
- Evidence gốc (trong repo TF4): `docs/evidence/mandate-04-forensic/`
