# So sánh giải pháp phân tích log cho CDO07 (sau Mandate-04)

> Mục tiêu so sánh: **làm audit dễ hơn** — query nhanh, UI, không phải down S3 mỗi lần —  
> trong khi **giữ S3 Object Lock làm source of truth bất biến**.  
> Volume tham chiếu: ~170 MB/ngày (~5.1 GB/tháng). Ngân sách chung TF ~$300/tuần.  
> **Kết luận chốt thành giải pháp:** xem [04-giai-phap-de-xuat.md](./04-giai-phap-de-xuat.md) · ADR [05](./05-ADR-chon-lop-phan-tich-log.md).

---

## 1. Nguyên tắc chọn (đừng nhầm với “thay thế logging”)

| Phải giữ | Được thêm |
|---|---|
| S3 Object Lock COMPLIANCE (CloudTrail + EKS) | Index / search / dashboard |
| Separation of Duties (operator không xóa audit) | Saved query cho drill ≤10 phút |
| CloudTrail log file validation | Một chỗ query nhiều nguồn (nếu được) |
| Identity mapping SSO → người | UI demo mentor (không chỉ CLI) |

**Anti-pattern:** Đưa audit log vào OpenSearch/ELK rồi coi đó là “bản chính” mà **tắt** Object Lock / Firehose. Index có thể bị xóa; S3 COMPLIANCE thì không.

---

## 2. Ma trận so sánh nhanh

| Tiêu chí | OpenSearch (đã có trong cluster) | ELK (Elastic Stack) | Grafana Loki | Datadog | Athena + S3 (+ Glue) |
|---|---|---|---|---|---|
| Mô hình | Full-text search / index | Full-text search / index | Label-indexed log store | SaaS observability | Query tại chỗ trên S3 (serverless) |
| UI sẵn | Grafana Explore + OpenSearch Dashboards | Kibana | Grafana Explore | Datadog UI | Athena console / QuickSight |
| Học phí vận hành | Trung bình (đã có pod) | Cao (stack nặng) | Thấp–TB nếu đã Grafana | Thấp (SaaS) | Thấp (SQL) |
| Chi phí ở ~5 GB/tháng | Chủ yếu compute/PVC trên EKS | Cao hơn OpenSearch tương đương | Thấp hơn full index | Đắt theo ingest/host | Pay-per-query — rất rẻ nếu ít audit |
| Query audit JSON sâu | Tốt (field mapping) | Tốt nhất hệ full-text | Trung bình (cần parse/labels) | Tốt | Tốt nếu partition + Glue catalog |
| Phù hợp “down S3 mỗi lần” | Cần ingest từ S3/CW | Cần ingest | Cần ingest (Promtail/Fluent Bit) | Agent / forwarder | **Không cần down** — query thẳng S3 |
| Risk với tamper-evident | Index xóa được → không thay S3 | Như OpenSearch | Như trên | Data rời AWS → compliance cần review | S3 vẫn SoT; Athena chỉ đọc |
| Fit TF4 hiện tại | **Cao** — đã có OS + Grafana | Thấp — trùng OpenSearch | Trung bình — stack Grafana sẵn | Thấp–TB — budget + vendor | **Cao** cho audit thưa / sâu S3 |
| Phụ thuộc nhóm khác | CDO08 (observability), CDO04 (infra) | CDO04 + CDO08 | CDO08 | CDO04 (billing/IAM) + Sec | CDO04 (Glue/IAM) |

---

## 3. Chi tiết từng phương án

### 3.1. OpenSearch (ưu tiên nghiên cứu sâu — vì đã có)

**Hiện trạng trong TF4:** namespace `techx-observability`, Grafana datasource `webstore-logs` → index `otel-logs-*` (app/OTel). Jaeger cũng dùng OpenSearch làm trace backend.

**Cách dùng cho forensic (đề xuất kiến trúc):**

```
S3 (Object Lock)  ──(optional batch / Lambda)──►  OpenSearch index audit-*
CloudWatch EKS    ──(Fluent Bit / subscription)─►  OpenSearch index eks-audit-*
CloudTrail        ──(đọc qua CW hoặc S3)────────►  OpenSearch index cloudtrail-*
                              │
                              ▼
                     Grafana Explore (saved queries forensic)
```

**Ưu:** Không thêm stack mới; team đã quen Grafana; full-text + field filter giúp cắt thời gian jq/CLI.  
**Nhược / rủi ro đã ghi trong scan CDO07:**

- `DISABLE_SECURITY_PLUGIN=true` — nếu ingest audit vào đây **bắt buộc** bật auth + RBAC (CDO08-SEC-03).
- Persistence / ISM retention phải rõ: index **không** thay Object Lock; ISM chỉ quản vòng đời bản copy search.
- Không trộn `otel-logs-*` với audit index (role/query/ACL khác nhau).

**Ước cost thô (order of magnitude):** với ~5 GB/tháng, chi phí chính là node EBS + CPU hiện có; ingest thêm ~170 MB/ngày thường chấp nhận được nếu PVC đủ. Cần CDO04/CDO08 đo lại PVC.

---

### 3.2. ELK (Elasticsearch + Logstash/Beats + Kibana)

**Bản chất:** Cùng họ full-text search với OpenSearch (OpenSearch fork từ ES).  
**Với TF4:** Gần như **trùng năng lực** OpenSearch đang chạy → mang thêm 1 cụm ES + Kibana là lãng phí ngân sách và vận hành.

**Khi nào mới cân nhắc ELK:** yêu cầu feature Elastic commercial cụ thể (SIEM license…) — **không thấy** trong Mandate-04 hiện tại.

**Kết luận nghiên cứu:** **Không đề xuất đưa ELK vào backlog** trừ khi có yêu cầu Elastic riêng. Ưu tiên OpenSearch hoặc Athena.

---

### 3.3. Grafana Loki

**Mô hình:** “Prometheus cho logs” — index theo **labels**, nội dung log nén rẻ; query LogQL trên Grafana.

**Ưu:** Nhẹ hơn full Elasticsearch; UI = Grafana đã có; cost storage thường thấp.  
**Nhược cho forensic audit JSON:**

- Audit event CloudTrail/EKS là JSON lớn (~150 KB/event theo báo cáo Mandate-4) — Loki không tối ưu kiểu “search mọi field tự do” như OpenSearch trừ khi parse kỹ lúc ingest.
- Drill kiểu “tìm mọi AccessDenied / mọi ConfigMap update theo field” cần pipeline extract labels/JSON — công sức không nhỏ hơn OpenSearch mapping.
- Vẫn cần **shipper** (Promtail / Fluent Bit / Alloy) từ CloudWatch/S3.

**Kết luận:** Phù hợp **app logs** giá rẻ; **không phải lựa chọn #1** cho CloudTrail/EKS forensic trừ khi CDO08 chuẩn hóa toàn bộ log về Loki và CDO07 chấp nhận LogQL + structured metadata.

---

### 3.4. Datadog

**Ưu:** UI mạnh, APM+logs+security cùng chỗ, onboarding nhanh, saved views/monitors đẹp cho demo.  
**Nhược với ràng buộc TF4:**

- Chi phí SaaS theo ingest/host thường **vượt** phần ngân sách còn lại nếu ingest đủ CloudTrail + EKS audit liên tục.
- Log rời khỏi AWS account → cần review compliance / DPA; mâu thuẫn tinh thần “S3 Object Lock trong account là SoT”.
- Vendor lock; forensic drill vẫn phải chứng minh đọc được từ audit trail “chính chủ” AWS khi mentor hỏi tamper-evident.

**Kết luận:** Có thể là **P3 / nice-to-have nghiên cứu** hoặc dùng **trial ngắn** so sánh UX — **không** đề xuất làm nền tảng forensic chính trong phase này.

---

### 3.5. Athena + Glue (+ tùy chọn CloudTrail Lake) — bổ sung quan trọng

Task liệt kê 4 cái trên thị trường; với volume nhỏ và pain “phải down S3”, **Athena** thường thắng về ROI:

| Hạng mục | Ghi chú |
|---|---|
| Mô hình | SQL trên file `.json.gz` / Parquet ngay trên S3 |
| Không thay Object Lock | Chỉ `SELECT` — SoT vẫn S3 |
| Pain “down về chạy code” | Biến thành query Athena / saved workgroup |
| Chi phí | ~$5/TB scanned; với partition `year/month/day/hour` + vài GB → **cents đến vài USD/tháng** nếu audit không liên tục |
| Hạn chế | Không real-time như live index; UI kém “sexy” hơn Kibana; cần Glue table / partition projection |

**CloudTrail Lake** (AWS): query SQL native CloudTrail — tiện nhưng có phí event data store; cân nhắc nếu chỉ cần CloudTrail (không cover EKS gzip trên S3 khác).

---

## 4. Mapping giải pháp → pain của task

| Pain task | OpenSearch | ELK | Loki | Datadog | Athena |
|---|---|---|---|---|---|
| Log nhiều chỗ, khó đọc | Gộp index + Grafana | Gộp + Kibana | Gộp + Grafana | Một UI SaaS | Nhiều table SQL; chưa “một UI” trừ QuickSight |
| Phải down S3 phân tích | Cần ingest trước | Cần ingest | Cần ingest | Cần forward | **Giải trực tiếp** |
| Drill ≤10 phút | Saved query giảm CLI/jq | Tương tự | Tốt nếu labels đúng | Tốt | Tốt nếu table sẵn + partition |
| Budget $300/tuần | Dùng lại stack | Đắt vận hành | Rẻ | Dễ đắt | Rẻ nhất khi ít query |
| Không phá tamper-evident | OK nếu S3 vẫn SoT | OK nếu S3 SoT | OK | Cần policy rõ | **Tốt nhất alignment** |

---

## 5. Khuyến nghị nghiên cứu (chưa phải quyết định triển khai)

### Phương án A — “Nhanh / rẻ / đúng pain S3” (đề xuất backlog P0 nghiên cứu PoC)

**Athena + Glue catalog** trên 2 bucket Object Lock (CloudTrail + EKS audit).  
PoC: 3 saved query tương ứng 3 drill scenario đã pass → đo thời gian so với CLI.

### Phương án B — “UX demo mentor + tái dùng CDO08” (P1)

**Ingest EKS audit (và/hoặc CloudTrail) vào OpenSearch index riêng `audit-*`**, query qua Grafana Explore;  
**đồng thời** giữ Athena cho truy vấn >7 ngày / scan rộng trên S3.

### Phương án C — Không làm trong phase này

- ELK riêng  
- Datadog làm SoT forensic  
- Loki làm primary CloudTrail store  

### Phương án kết hợp thực dụng (khuyến nghị đưa backlog)

```
S3 Object Lock  =  nguồn sự thật bất biến (đã có)
Athena          =  audit sâu / >7 ngày / không down file
OpenSearch+Grafana =  drill nhanh 7 ngày gần + demo UI (sau khi sec plugin + ACL)
```

---

## 6. Câu hỏi còn mở (cần làm rõ trước khi estimate story point)

1. Mentor chấp nhận demo qua **Grafana** hay bắt buộc thấy **raw CLI trên AWS**? (ảnh hưởng độ ưu tiên UI)
2. CDO08 có roadmap bật **OpenSearch security plugin + ISM** không? (blocker nếu ingest audit)
3. Có cần **correlate** audit trail với app log `otel-logs-*` / Jaeger trong cùng dashboard không?
4. Tần suất audit: chỉ drill/sự cố hay **SOC-like** hàng ngày? (Athena thắng nếu thưa; index thắng nếu dày)
5. Ai owns vận hành index: CDO07 (người dùng) vs CDO08 (platform observability)?

Trả lời 5 câu này trong backlog grooming sẽ chốt A vs B vs A+B.
