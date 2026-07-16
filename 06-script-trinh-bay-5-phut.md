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

## 1:45–3:15 — So sánh nhanh 4 giải pháp thị trường

“Nhóm so sánh OpenSearch, ELK, Loki, Datadog, và thêm Athena vì đúng pain S3.

- **ELK:** trùng OpenSearch đang có → loại.
- **Datadog:** dễ đắt, data ra ngoài account, không làm SoT → loại phase này.
- **Loki:** tốt app log giá rẻ; kém fit JSON CloudTrail lớn → hoãn.
- **OpenSearch:** có sẵn Grafana; nhưng security plugin đang tắt → chỉ P2 sau khi harden.
- **Athena:** query thẳng S3, không download, rẻ, giữ Object Lock → **chọn P0**.”

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
