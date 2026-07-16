# Hiện trạng Mandate-04 Forensic — Tóm tắt cho task "Phân tích giải pháp phân tích log"

> **Nhóm:** CDO07 (Auditability) · **Ngày:** 16/07/2026  
> **Thư mục:** `C:\Users\NguyenPhuTrieu\Downloads\research-cdo07-log-analysis`  
> **Nguồn trong repo:** `mandates/MANDATE-04-auditability-tf4.md`,
> `docs/evidence/mandate-04-forensic/*`, `docs/audit/tickets/AUDIT-CDO07-MANDATE04-FORENSIC-FRAMEWORK.md`,
> `docs/audit/runbooks/forensic-playbook-timeline.md`
>
> **Đọc kèm:** [00](./00-ban-do-mandate-04-forensic.md) · [02](./02-so-sanh-giai-phap-phan-tich-log.md) ·
> [03](./03-de-xuat-backlog.md) · **giải pháp [04](./04-giai-phap-de-xuat.md)** · ADR [05](./05-ADR-chon-lop-phan-tich-log.md)

---

## 1. Mandate-04 yêu cầu gì

Directive #4 (chỉ áp dụng cho TF4) — Ban Kiểm toán không hỏi "hệ thống có chạy không" mà hỏi:
**"chuyện gì đã xảy ra, ai làm, khi nào — và làm sao tin được bản ghi đó?"**

4 yêu cầu chính:

1. **Đường ghi vết đầy đủ** — K8s audit log + CloudTrail + change trail (ai/khi nào/nội dung).
2. **Bài forensic chấm tại chỗ** — mentor chọn 1 sự kiện, TF4 dựng lại timeline
   ai-làm-gì-khi-nào **chỉ từ audit log**, trong thời gian giới hạn (≤10 phút/scenario).
3. **Tamper-evident** — log không sửa/xóa được tùy tiện; quyền ghi tách khỏi người vận hành.
4. **Truy về người** — mọi thay đổi lớn + hành động on-call quy về 1 danh tính cá nhân
   (không dùng shared account).

Ràng buộc: ngân sách **~$300/tuần/TF**; không đụng flagd; storefront công khai, cổng vận hành riêng tư.

## 2. Kiến trúc audit logging hiện tại (đã triển khai xong, evidence 15/07/2026)

> Sync repo 16/07: CloudTrail **có** CloudWatch Logs delivery; AWS Config recording; xem `07-sync-repo-2026-07-16.md`.

```
K8s API Server (EKS techx-tf4-cluster)
   │  Control Plane Logging (api, audit, authenticator, controllerManager, scheduler)
   ▼
CloudWatch Logs  /aws/eks/techx-tf4-cluster/cluster
   │  Subscription Filter "eks-audit-to-firehose"
   ▼
Kinesis Data Firehose  tf4-eks-audit-logs-firehose    (ACTIVE; GZIP)
   ▼
S3  tf4-eks-audit-logs-511825856493
    └─ Object Lock COMPLIANCE 90 ngày + Versioning + SSE

AWS API calls
   ▼
CloudTrail  tf4-general-cloudtrail
   (multi-region, LogFileValidationEnabled=true, digest SHA-256)
   ├──► CloudWatch Logs  /aws/cloudtrail/tf4-general-cloudtrail   ← dùng Insights
   └──► S3  tf4-cloudtrail-logs-bucket-511825856493
            prefix AWSLogs/511825856493/CloudTrail/
            └─ Object Lock COMPLIANCE 90 ngày + Versioning + SSE

AWS Config  tf4-config-recorder  (recording=true, delivery Success)
```

Bảo vệ tamper-evident (đã test, evidence AUD-17.3):

- **S3 Object Lock COMPLIANCE 90 ngày** — cả admin lẫn root không xóa được object trước hạn (test DeleteObject → AccessDenied).
- **Separation of Duties** — role `TF4-Developer` (operator) bị Explicit Deny toàn bộ
  DeleteObject / StopLogging / DeleteLogGroup (4/4 test AccessDenied).
- **CloudTrail Log File Validation** — digest SHA-256; `validate-logs` pass 34/34 digest, 127/127 log files.
- **Firehose chống root-deletion** — kể cả khi CloudWatch Log Group bị xóa, bản sao trên S3 vẫn bất biến.

Identity mapping (AUD-17.4): 100% thành viên 4 nhóm (CDO04/07/08, AIO01) map về SSO
permission set cá nhân, không có shared account; session name = username nên CloudTrail
trace được về người thật.

## 3. Quy trình forensic hiện tại — và đây chính là chỗ đau

Cách truy vết hiện nay (theo `forensic-playbook-timeline.md` + drill log AUD-17.2):

1. Phân loại sự kiện: trong cluster → K8s Audit Log; AWS API → CloudTrail.
2. Tự tính **epoch milliseconds** cho time window.
3. Chạy `aws logs filter-log-events` hoặc `aws cloudtrail lookup-events` bằng CLI.
4. Pipe qua **jq** để parse JSON, lọc verb/user/resource.
5. Đọc output text, tự dựng bảng WHO/WHAT/WHEN/HOW.

Kết quả drill (3/3 pass): 6m50s / 9m05s→6m40s / 6m20s — pass nhưng **sát giới hạn 10 phút**,
và bản thân drill log đã tự ghi nhận các điểm nghẽn:

| Điểm nghẽn ghi nhận trong drill | Bản chất vấn đề |
|---|---|
| Tính epoch ms thủ công (~45s/lần) | Không có UI query theo thời gian |
| jq syntax `fromjson` dễ gõ nhầm | Không có ngôn ngữ query thân thiện |
| CloudWatch filter pattern phải thử 2–3 lần | Full-text filter thô, không đánh index theo field |
| Noise từ service account phải lọc bằng jq | Không có filter field-level sẵn |
| `lookup-events` chậm (~1.5x) | CloudTrail API chỉ tra được 90 ngày, throttle 2 req/s |

## 4. Vấn đề của task hiện tại (đúng như mô tả task)

1. **Log rải rác nhiều chỗ:** CWL EKS, CWL CloudTrail, CloudTrail API/`lookup-events`, S3 CT, S3 EKS, AWS Config, app log OpenSearch (`otel-logs-*`).
2. **Audit log trên S3 chỉ đọc được bằng cách down về + chạy code phân tích:** file `.json.gz` — đúng pain task.
3. **Không có quy trình phân tích chuẩn hóa:** playbook vẫn CLI-first; Insights được gợi ý nhưng chưa có saved query / dashboard forensic.
4. **OpenSearch không sẵn sàng làm lớp audit:** security plugin tắt; PVC 8Gi đã từng chạm watermark (CDO08 incident) — không nên đổ thêm audit vào.
5. **Docs drift:** playbook còn ghi `LogFileValidationEnabled=false` dù evidence đã PASS `true`.

→ Cần lớp phân tích đặt LÊN TRÊN kho bất biến; **không thay** Object Lock.  
→ Gắn với Task 3.1 (ISM) + Task 3.2 (Grafana Audit Dash) đã có trong kế hoạch CDO07.

## 5. Số liệu đầu vào cho bài toán chọn giải pháp

| Thông số | Giá trị | Nguồn |
|---|---|---|
| Dung lượng nạp log | **~170 MB/ngày** (~5.1 GB/tháng) | Báo cáo nghiên cứu Mandate-4 |
| Kích thước event trung bình | ~150 KB | Báo cáo nghiên cứu Mandate-4 |
| Chi phí luồng log hiện tại | ~$0.64/tuần (<1% ngân sách) | Báo cáo nghiên cứu Mandate-4 |
| Ngân sách còn lại | ~$300/tuần/TF (dùng chung mọi thứ) | Mandate-04 |
| Retention bất biến | 90 ngày (Object Lock COMPLIANCE) | AUD-17.3 |
| Region | us-east-1 | evidence |
| CloudTrail CWL group | `/aws/cloudtrail/tf4-general-cloudtrail` | AUD-17.1 |
| EKS CWL group | `/aws/eks/techx-tf4-cluster/cluster` | AUD-17.1 |
| Đã có trong cluster | Grafana + Jaeger + OpenSearch (PVC 8Gi, disk pressure) | CDO08 week2 |
| Backlog sẵn | Task 3.1 ISM, Task 3.2 Grafana Audit Dash | DELEGATED_TASKS_P0 / TEAM_ASSIGNMENT |
| Tần suất audit | Không liên tục — theo drill/sự cố/đợt kiểm toán | drill log |
