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

### P0 — Nghiên cứu & quyết định (task hiện tại của bạn)

#### CDO07-LOG-01 — Phân tích hiện trạng nguồn log & pain forensic
- **Type:** Research  
- **Estimate:** 0.5–1 SP  
- **DoD:**
  - [ ] Sơ đồ nguồn log (CloudWatch / CloudTrail / S3 CT / S3 EKS / otel-logs) — xem file `00` + `01`
  - [ ] Liệt kê pain từ drill log AUD-17.2 (epoch, jq, multi-source, S3 download)
  - [ ] Chốt nguyên tắc: index/UI **không** thay Object Lock
- **Output:** file `01` (đã có) + trình bày nhóm

#### CDO07-LOG-02 — So sánh OpenSearch / ELK / Loki / Datadog (+ Athena)
- **Type:** Research  
- **Estimate:** 1–2 SP  
- **DoD:**
  - [ ] Ma trận tiêu chí (cost, UX, tamper-evident, fit stack hiện có)
  - [ ] Khuyến nghị A/B/C có lý do loại ELK/Datadog (nếu giữ nguyên khuyến nghị file `02`)
  - [ ] Danh sách câu hỏi mở cho grooming với CDO04/CDO08
- **Output:** file `02` (đã có) → cập nhật sau khi hỏi mentor/CDO08

#### CDO07-LOG-03 — ADR / Decision record chọn hướng
- **Type:** Decision  
- **Estimate:** 0.5 SP  
- **DoD:**
  - [ ] Chọn: Athena-only / OpenSearch-only / **Athena + OpenSearch** / khác
  - [ ] Owner vận hành (CDO07 vs CDO08) ghi rõ
  - [ ] Cost ceiling tuần (vd. ≤ $X trong $300)
- **Phụ thuộc:** trả lời câu hỏi mở ở file `02` §6

---

### P1 — PoC (sau khi ADR chốt)

#### CDO07-LOG-04 — PoC Athena trên bucket audit Object Lock
- **Type:** Spike / Lab  
- **Estimate:** 2 SP  
- **Assignee gợi ý:** CDO07 + hỗ trợ IAM từ CDO04  
- **DoD:**
  - [ ] Glue table (hoặc partition projection) cho `tf4-cloudtrail-logs-bucket-*` và/hoặc `tf4-eks-audit-logs-*`
  - [ ] ≥3 saved query map 3 drill scenario đã pass (AUD-17.2)
  - [ ] Đo thời gian: Athena vs CLI playbook (bảng số)
  - [ ] Evidence PoC trong thư mục research hoặc `docs/evidence/` (khi nhóm chốt)
  - [ ] Xác nhận: query **không** cần `s3:DeleteObject`; chỉ read
- **Delegated (nếu thiếu quyền):** ticket kiểu AUDIT-010 bổ sung `athena:*` / `glue:Get*` read-only cho `TF4-AuditReadOnlyAndAnalyze`

#### CDO07-LOG-05 — PoC Grafana saved queries cho EKS audit (7 ngày gần)
- **Type:** Spike  
- **Estimate:** 1–2 SP  
- **Phụ thuộc:** CDO08 (datasource / quyền Explore)  
- **DoD:**
  - [ ] Ít nhất 2 query mẫu: ConfigMap update, port-forward / secrets-related
  - [ ] Document: dùng CloudWatch Logs datasource **hoặc** OpenSearch index riêng (nếu đã ingest)
  - [ ] So sánh thời gian với `aws logs filter-log-events` + jq
- **Không làm nếu:** ADR chọn Athena-only thuần và mentor không cần UI

---

### P2 — Hardening nếu chọn OpenSearch làm lớp search

#### CDO07-LOG-06 — Ingest audit trail vào OpenSearch index `audit-*` (không trộn otel-logs)
- **Type:** Story  
- **Estimate:** 3–5 SP  
- **Phụ thuộc cứng:**
  - CDO08: bật security plugin / ACL (liên quan SEC-03)
  - CDO08/CDO04: PVC + ISM retention cho index audit
- **DoD:**
  - [ ] Pipeline ingest (Fluent Bit / Lambda) từ CW hoặc S3 → index `audit-*`
  - [ ] RBAC: chỉ nhóm audit đọc được index
  - [ ] Runbook: “khi index lệch S3 thì tin S3 + Athena”
  - [ ] Drill rehearsal ≤10 phút trên UI

#### CDO07-LOG-07 — Đồng bộ runbook forensic (CLI → UI/Athena)
- **Type:** Docs  
- **Estimate:** 1 SP  
- **DoD:**
  - [ ] Cập nhật `forensic-playbook-timeline.md` (hoặc bản song song) với đường query mới
  - [ ] Mentor runbook: 1 trang “cách xem log” theo hướng đã chốt
  - [ ] Giữ lệnh CLI làm fallback khi UI down

---

### P3 — Không ưu tiên phase này (ghi rõ để tránh scope creep)

| ID | Item | Lý do để sau |
|---|---|---|
| CDO07-LOG-08 | Triển khai ELK riêng | Trùng OpenSearch; tốn vận hành |
| CDO07-LOG-09 | Datadog làm forensic SoT | Cost + data rời account |
| CDO07-LOG-10 | Loki primary cho CloudTrail | JSON audit lớn; công sức parse cao |
| CDO07-LOG-11 | Correlate full app log ↔ audit 1 dashboard | Nice-to-have; phụ thuộc chuẩn hóa OTel của CDO08 |

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
