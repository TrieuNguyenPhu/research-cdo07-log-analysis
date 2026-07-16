# Tối ưu chi phí lớp phân tích log (CDO07)

> Dựa trên volume ~170 MB/ngày (~5.1 GB/tháng), audit **thưa** (drill/sự cố), ngân sách TF ~$300/tuần.  
> Luồng ghi+khóa hiện tại đã ~$0.64/tuần — giữ nguyên, **không** nhân đôi ingest.

---

## Kết luận một dòng

**Rẻ nhất đúng pain:** `CLI sẵn có (miễn phí thêm)` → **CloudWatch Logs Insights (P1)** cho sự kiện gần → **Athena (P0)** khi cần đọc S3 / ngoài cửa sổ CWL.  
**Đắt nhất / tránh:** Datadog, ELK mới, ingest thêm audit vào OpenSearch (đã đầy đĩa).

---

## Xếp hạng cost (ước lượng us-east-1)

| Hạng | Phương án | Mô hình tính tiền | Ước với ~20 query forensic/tuần | Ghi chú |
|---|---|---|---|---|
| 1 | Giữ CLI (`lookup-events` / `filter-log-events`) | Gần như $0 thêm | **~$0** | Chậm UX; đã có |
| 2 | **Athena** trên S3 Object Lock | $5/TB scanned (~$0.005/GB) | **~$0.01–0.50/tuần** nếu partition theo ngày | Đúng pain “down S3” |
| 3 | **Logs Insights** trên CWL sẵn có | $0.005/GB scanned | **~$0.05–1/tuần** nếu hẹp time range | CT + EKS đã có log group |
| 4 | Grafana Explore → CloudWatch | Như Insights + (dashboard free trong quota) | Tương đương Insights | Extend Task 3.2 |
| 5 | OpenSearch ingest thêm audit | EBS + CPU **luôn chạy** + risk tăng PVC 8→20Gi | **+$ vài–vài chục $/tháng** cố định | Đang watermark — đắt & rủi |
| 6 | Loki / ELK mới | Node luôn chạy | Cao hơn tái dùng OS | Không cần |
| 7 | Datadog | Theo ingest/host | Dễ **vượt** phần budget còn lại | Loại |

Số Athena: scan 2 GB/query × 20 query ≈ 40 GB ≈ 0.04 TB × $5 ≈ **$0.20/tuần** (trường hợp xấu nếu quên partition). Có partition ngày + filter chặt → thường **cents**.

---

## Stack tối ưu cost đề xuất (làm theo thứ tự)

```
SoT S3 Object Lock     ← giữ (đã trả ~$0.64/tuần cho pipeline)
        │
        ├─ drill gần     → CloudWatch Logs Insights / Grafana (pay per scan)
        └─ audit sâu/S3  → Athena (pay per scan)   ★ không download, không cluster mới
```

**Không làm để tiết kiệm:**

1. Không ingest CloudTrail/EKS audit vào OpenSearch (nhân đôi lưu trữ + đẩy PVC).
2. Không bật Datadog/ELK cho forensic.
3. Không kéo dài retention CloudWatch chỉ để “query dễ” — CWL ingest $0.50/GB đắt hơn để S3 + Athena.
4. Không chạy scheduled Insights/Athena rộng (full log group, 30 ngày) mỗi giờ.

---

## Thủ thuật cắt bill (bắt buộc ghi vào PoC)

### Athena

- Partition / path filter: `year=…/month=…/day=…` hoặc `WHERE` theo ngày trước khi `SELECT *`.
- Workgroup `tf4-cdo07-audit`: **limit bytes scanned / query** (vd. 5–10 GB).
- Result bucket lifecycle **7 ngày** (đừng Object Lock result).
- Ưu tiên query cột cần thiết; về sau nếu ổn định có thể convert Parquet (không bắt buộc PoC).

### Logs Insights

- Luôn set **time window hẹp** (phút–giờ của sự kiện mentor), không mặc định 1 tuần.
- Không schedule query rộng.
- Dùng log group đúng (`/aws/cloudtrail/...` hoặc `/aws/eks/...`), không quét nhiều group cùng lúc nếu không cần.

### CloudWatch retention (cân nhắc với CDO04 — trade-off)

- Giữ retention ngắn cho CWL (cửa sổ drill) + SoT dài trên S3 = **rẻ hơn** tăng retention CWL lên 90 ngày.
- Firehose → S3 đã trả một lần; đừng giữ 2 bản “nóng” lâu trên CWL nếu chỉ để forensic thưa.

---

## Ceiling đề xuất đưa ADR / backlog

| Mục | Ceiling |
|---|---|
| Lớp phân tích (Athena + Insights) PoC | **≤ $5/tuần** |
| Cảnh báo | Athena workgroup limit + Cost Explorer filter `Athena` / `CloudWatch` |
| So sánh | Không vượt thêm **1–2%** ngân sách $300/tuần |

Thực tế với kỷ luật partition + time range: kỳ vọng **<$1/tuần**.

---

## Trả lời nhanh khi bị hỏi “sao không OpenSearch cho rẻ?”

OpenSearch **đã trả** cho app log/traces; thêm audit = thêm disk (CDO08 đang đề xuất 20Gi) + CPU 24/7 trong khi CDO07 chỉ query thưa.  
Pay-per-query (Athena/Insights) khớp pattern “ít dùng, cần đúng lúc” → **rẻ hơn TCO** dù “có sẵn” UI OpenSearch.
