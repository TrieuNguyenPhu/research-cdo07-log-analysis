# Phụ lục: So sánh giải pháp & cost

> Đọc khi họp bị hỏi “sao không dùng OpenSearch / ELK / Loki / Datadog?”.  
> Phương án chính: [01-phuong-an-de-xuat.md](./01-phuong-an-de-xuat.md).

---

## 1. Nguyên tắc

| Phải giữ | Được thêm |
|---|---|
| S3 Object Lock COMPLIANCE (SoT) | Index / query / UI |
| Separation of duties | Saved query forensic |
| CloudTrail log file validation | Không thay thế S3 bằng index |

**Anti-pattern:** Đưa audit vào OpenSearch rồi coi đó là bản chính.

---

## 2. Ma trận so sánh

| Tiêu chí | OpenSearch (có sẵn) | ELK | Loki | Datadog | **Athena + Insights** |
|---|---|---|---|---|---|
| Giải “phải down S3” | Cần ingest trước | Cần ingest | Cần ingest | Forwarder | **Athena query thẳng S3** |
| Cost (audit thưa, ~5 GB/tháng) | Phí node/EBS cố định | Cao hơn | Trung bình | Dễ đắt | **Pay-per-query — rẻ nhất** |
| Tamper-evident | Index xóa được | Như OS | Như OS | Data ngoài account | **S3 vẫn SoT** |
| Fit TF4 hiện tại | UI sẵn nhưng **sec tắt + disk đầy** | Trùng OS | Grafana sẵn | Vendor mới | CT/EKS CWL + S3 đã có |
| Thời gian PoC | Chờ CDO08 harden | Chậm | Trung bình | Trung bình | **Nhanh (IAM + Glue)** |
| **Quyết định** | **P2 hoãn** | **Loại** | **Hoãn** | **Loại** | **Chọn** |

### Ghi chú OpenSearch (tại sao không chọn ngay)

- ADR-009: `DISABLE_SECURITY_PLUGIN=true`.
- CDO08: PVC 8Gi đã watermark / Jaeger không ghi được; đề xuất tăng 20Gi.
- Đang chứa app log + traces — ingest thêm CloudTrail/EKS = nhân đôi lưu + rủi vận hành.

---

## 3. Cost chi tiết (us-east-1, order of magnitude)

| Phương án | Mô hình | ~20 query forensic/tuần |
|---|---|---|
| CLI hiện tại | ~$0 thêm | $0 (nhưng UX kém) |
| **Athena** | $5/TB scanned | Thường cents; xấu ~$0.20 nếu quên partition |
| **Logs Insights** | $0.005/GB scanned | Thấp nếu hẹp time window |
| OpenSearch + ingest audit | EBS/CPU 24/7 | +vài–vài chục $/tháng cố định |
| Datadog | Theo ingest | Dễ vượt phần budget còn lại |

Pipeline ghi hiện tại (giữ): ~$0.64/tuần.

**Ceiling đề xuất:** lớp phân tích **≤ $5/tuần** · kỳ vọng **<$1/tuần**.

### Cách giữ bill thấp

- Athena: filter/partition theo ngày; workgroup limit bytes/query; result lifecycle ngắn.
- Insights: time window hẹp (phút–giờ sự kiện); không schedule query rộng.
- Không tăng retention CWL lên 90 ngày chỉ để query — để S3 + Athena.
- Không ingest audit vào OpenSearch giai đoạn này.

---

## 4. Hiện trạng nguồn log (tóm tắt)

```
CloudTrail → S3 (Object Lock) + CWL /aws/cloudtrail/tf4-general-cloudtrail
EKS audit  → CWL /aws/eks/.../cluster → Firehose → S3 (Object Lock, year=/month=/day=/hour=)
AWS Config → staging + WORM archive (change trail)
App logs   → OpenSearch otel-logs-* (CDO08) — khác bài toán forensic Mandate-04
```

Chi tiết DDL/Athena: [05-architecture-athena-forensics.md](./05-architecture-athena-forensics.md).

Volume tham chiếu: ~170 MB/ngày EKS path (~5.1 GB/tháng) + CloudTrail/Config nhỏ hơn.

Với tần suất audit **thưa** và SoT đã nằm trên S3: **Athena + Insights** thắng về cost, tốc độ PoC và alignment Mandate-04. Các giải pháp full-index/SaaS phù hợp hơn khi cần SOC liên tục — không phải pattern hiện tại của CDO07.
