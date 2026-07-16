# Script trình bày ~5 phút (họp backlog / sync nhóm)

> Đọc to được; chỉnh tên người cho phù hợp.

---

## 0:00–0:45 — Bối cảnh Mandate-04

“Mandate-04 yêu cầu dựng lại ai-làm-gì-khi-nào từ audit log trong ≤10 phút, và chứng minh log không sửa xóa được.

Phần **ghi và khóa** đã xong: CloudTrail và EKS audit đổ S3 Object Lock COMPLIANCE 90 ngày, Firehose chống xóa trên CloudWatch, drill 3/3 pass. Chi phí luồng log dưới 1 đô/tuần.”

## 0:45–1:45 — Pain còn lại (= task này)

“Cái còn đau đúng như task mô tả:

Một — log nằm nhiều chỗ: CloudWatch, CloudTrail API, hai bucket S3, app log OpenSearch của CDO08.

Hai — audit sâu phải download S3 rồi chạy code/script.

Ba — hiện forensic chủ yếu CLI + jq, sát 10 phút, khó người mới và khó demo bền.

→ Cần lớp **phân tích**, không phải làm lại lớp ghi.”

## 1:45–3:15 — So sánh nhanh

“OpenSearch đã có nhưng **security tắt** và CDO08 vừa gặp **đầy đĩa PVC 8Gi** — không ingest audit vào đó lúc này.

ELK trùng OS → loại. Datadog đắt/egress → loại. Loki kém fit JSON CloudTrail → hoãn.

**Athena** query thẳng S3 → đúng pain download → P0.  
Đồng thời CloudTrail **đã** đổ CloudWatch — dùng **Logs Insights** cho drill gần (EKS + CloudTrail), gắn Task 3.2 Grafana Audit Dash sẵn có.”

## 3:15–4:15 — Giải pháp đề xuất

“Kiến trúc đề xuất:

- S3 Object Lock vẫn là source of truth.
- P0: Athena + Glue trên hai bucket audit.
- P1: CloudWatch Insights hoặc Grafana cho EKS trong 7 ngày — giảm jq.
- P2: OpenSearch index `audit-*` chỉ khi CDO08 bật auth.

PoC thành công khi một scenario drill chạy bằng Athena **không** cần `aws s3 cp`, và đo được thời gian giảm so với CLI.”

## 4:15–5:00 — Xin quyết định / next step

“Cần nhóm chốt ADR trong file `05`.

Hỏi CDO04: IAM Athena/Glue read-only có làm được tuần này không.  
Hỏi CDO08: Grafana Explore OK chứ; OpenSearch sec plugin có roadmap không.

Sau khi ADR Accepted, paste stories từ file `03` vào Jira: research đóng, PoC Athena mở.”

---

## Câu hỏi dự phòng

| Nếu bị hỏi… | Trả lời ngắn |
|---|---|
| Sao không dùng OpenSearch luôn? | Đang tắt security plugin; index xóa được — không thay Object Lock. Làm P2 sau. |
| Athena có real-time không? | Đủ cho audit theo sự cố/drill; sự kiện gần dùng CloudWatch. |
| Có phá Mandate-04 không? | Không — chỉ thêm lớp đọc; không tắt Firehose/Object Lock. |
| Budget? | Ceiling PoC đề xuất ≤ $5/tuần; volume ~5 GB/tháng. |
