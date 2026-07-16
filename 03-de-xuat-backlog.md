# Đề xuất backlog — Lớp phân tích log phục vụ audit (CDO07)

> Đầu vào: Mandate-04 đã có ghi + khóa log; task mới = làm **đọc/audit dễ hơn**.  
> So sánh: [02](./02-so-sanh-giai-phap-phan-tich-log.md) · Giải pháp chốt đề xuất: [04](./04-giai-phap-de-xuat.md) · ADR: [05](./05-ADR-chon-lop-phan-tich-log.md)  
> Thư mục: `C:\Users\NguyenPhuTrieu\Downloads\research-cdo07-log-analysis`

---

## Epic đề xuất

**Epic:** `[CDO07] Centralized Audit Log Analysis Layer`  
**Mục tiêu:** Auditor dựng WHO/WHAT/WHEN/HOW từ audit trail **không cần** `aws s3 cp` + script ad-hoc; drill trung bình rõ ràng dưới 10 phút; S3 Object Lock vẫn là source of truth.

**Out of scope epic này:** Thay CloudTrail / bỏ Firehose / đổi Object Lock; viết lại app logging của CDO08.

---

## Backlog items (gợi ý Jira)

> **Gắn với kế hoạch sẵn trong repo** (không tạo epic “trên trời”):
> - `docs/audit/DELEGATED_TASKS_P0.md` — Task 3.1 ISM, Task 3.2 Grafana Audit Dashboard  
> - `docs/audit/TEAM_ASSIGNMENT.md` — Member 7 Log Retention, Member 8 Dashboard  
> - `docs/audit/JIRA_TASKS.md` — Task 7–8  
> Sync chi tiết: [07-sync-repo-2026-07-16.md](./07-sync-repo-2026-07-16.md)

### P0 — Nghiên cứu & quyết định (task hiện tại của bạn)

#### CDO07-LOG-01 — Phân tích hiện trạng nguồn log & pain forensic
- **Type:** Research  
- **Estimate:** 0.5–1 SP  
- **DoD:**
  - [ ] Sơ đồ nguồn: CWL EKS, CWL CloudTrail, S3 CT, S3 EKS, Config, OpenSearch app
  - [ ] Pain từ AUD-17.2 + “phải down S3”
  - [ ] Chốt: index/UI **không** thay Object Lock
- **Output:** file `00`/`01` + trình bày nhóm

#### CDO07-LOG-02 — So sánh OpenSearch / ELK / Loki / Datadog (+ Athena)
- **Type:** Research · **Estimate:** 1–2 SP  
- **DoD:** ma trận + khuyến nghị; nêu OpenSearch disk pressure + security tắt là blocker ingest  
- **Output:** file `02`

#### CDO07-LOG-03 — ADR chọn hướng
- **Type:** Decision · **Estimate:** 0.5 SP  
- **DoD:** chốt Athena P0 + Insights P1 + OS P2; owner; cost ceiling  
- **Output:** file `05` Signed

#### CDO07-LOG-03b — Sửa drift playbook forensic (docs debt)
- **Type:** Docs · **Estimate:** 0.25 SP  
- **DoD:** `forensic-playbook-timeline.md` bỏ đoạn `LogFileValidationEnabled=false`; cập nhật Object Lock + validate-logs + gợi ý Insights/Athena  
- **Nguồn lệch:** playbook vs `aud-17.1` / `aud-17.3`

---

### P1 — PoC (sau ADR)

#### CDO07-LOG-04 — PoC Athena trên bucket Object Lock
- **Type:** Spike · **Estimate:** 2 SP  
- **DoD:** Glue/table cho `tf4-cloudtrail-logs-bucket-*` (prefix `AWSLogs/511825856493/CloudTrail/`) ± EKS bucket; ≥3 query map AUD-17.2; không `s3 cp`; đo thời gian vs CLI  
- **IAM:** bổ sung Athena/Glue read-only cho `TF4-AuditReadOnlyAndAnalyze`

#### CDO07-LOG-05 — Saved CloudWatch Logs Insights (EKS + CloudTrail CWL)
- **Type:** Spike · **Estimate:** 1 SP  
- **DoD:** ≥2 query EKS (`/aws/eks/techx-tf4-cluster/cluster`) + ≥1 query CloudTrail (`/aws/cloudtrail/tf4-general-cloudtrail`); so thời gian với CLI  
- **Note:** CloudTrail→CWL đã confirmed AUD-17.1 — ưu tiên cao hơn bản research cũ (chỉ EKS)

#### CDO07-LOG-05b — Extend Task 3.2 Grafana Audit Dashboard (đã có trong plan)
- **Type:** Story (nghiệm thu / requirement) · **Estimate:** 1–2 SP  
- **Liên kết:** DELEGATED Task 3.2 / JIRA Task 8 / Member 8  
- **DoD hiện có:** panel 401/403  
- **DoD mở rộng đề xuất:** link Explore tới Insights queries forensic; không bắt buộc ingest OpenSearch

---

### P2 — OpenSearch (sau capacity + security)

#### CDO07-LOG-06 — Chỉ mở khi CDO08: PVC≥20Gi (hoặc tương đương) + security plugin + ISM
- **Liên kết:** DELEGATED Task 3.1 / JIRA Task 7 / Member 7  
- **DoD:** ISM policy reviewed; index `audit-*` tách `otel-logs-*`; ACL; runbook “tin S3 khi lệch”  
- **Không làm nếu** disk vẫn watermark / plugin vẫn tắt

#### CDO07-LOG-07 — Đồng bộ runbook forensic (CLI → Insights/Athena/Grafana)
- **Type:** Docs · **Estimate:** 1 SP  
- **DoD:** playbook + mentor runbook cập nhật; CLI giữ fallback

---

### P3 — Không ưu tiên phase này

| ID | Item | Lý do |
|---|---|---|
| CDO07-LOG-08 | ELK riêng | Trùng OpenSearch |
| CDO07-LOG-09 | Datadog SoT | Cost + egress |
| CDO07-LOG-10 | Loki primary CloudTrail | Fit kém JSON audit |
| CDO07-LOG-11 | Ingest audit vào OpenSearch ngay | Disk pressure + no auth |

---

## Thứ tự làm việc đề xuất (2 tuần kiểu)

```
Tuần 1 (research — task hiện tại)
  LOG-01 hiện trạng  →  LOG-02 so sánh  →  họp CDO08/CDO04  →  LOG-03 ADR

Tuần 2 (nếu ADR = Athena + optional Grafana)
  LOG-04 PoC Athena  →  (optional) LOG-05 Grafana  →  demo nhóm / mentor
  Sau đó mới estimate LOG-06 nếu cần OpenSearch ingest
```

---

## Definition of Done cấp Epic (khi nào coi “xong bài toán task”)

1. Auditor có **ít nhất một đường** phân tích log trên S3 **không** cần download thủ công (Athena hoặc tương đương).
2. Có **decision record** loại trừ / hoãn ELK & Datadog (hoặc giải thích nếu chọn khác).
3. Drill mẫu lặp lại được với thời gian đo được; ideally trung bình rõ dưới ngưỡng 10 phút.
4. Tài liệu khẳng định: **S3 Object Lock vẫn là SoT**; lớp phân tích chỉ là bản copy/query.
5. Backlog P2 (OpenSearch ingest) chỉ mở khi security plugin + owner vận hành đã chốt.

---

## Câu slide / đoạn trình bày 60 giây (copy dùng họp)

> Mandate-04 đã xong phần **ghi và khóa** audit log: CloudTrail + EKS audit đổ S3 Object Lock COMPLIANCE, drill forensic 3/3 pass bằng CLI.  
> Pain còn lại: log nằm nhiều chỗ, audit sâu phải down S3 rồi chạy script — sát giới hạn 10 phút.  
> Task này nghiên cứu lớp **phân tích** (OpenSearch / ELK / Loki / Datadog, kèm Athena vì query thẳng S3).  
> Hướng đề xuất đưa backlog: **Athena PoC trước** (rẻ, đúng pain download), **OpenSearch/Grafana sau** nếu cần UI — không thay Object Lock, không dựng ELK trùng stack.

---

## File cần mang vào buổi backlog grooming

1. `00-ban-do-mandate-04-forensic.md` — context Mandate  
2. `02-so-sanh-giai-phap-phan-tich-log.md` — bảng so sánh  
3. File này — danh sách story + DoD  
4. (tuỳ chọn) `aud-17.2-drill-log.md` — số liệu thời gian drill thật làm baseline
