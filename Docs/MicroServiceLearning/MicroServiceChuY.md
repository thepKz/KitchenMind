# Cấu hình README & Checklist Microservice

Tài liệu này là checklist và hướng dẫn cấu hình thực tiễn cho kiến trúc microservice, dùng làm README hoặc tài liệu onboarding cho team.

---

## Nguyên tắc cốt lõi

- **Một service, một database**  
  Mỗi service sở hữu riêng database của mình. Không chia sẻ DB giữa các service. Mọi truy cập dữ liệu phải đi qua API hoặc event của service chủ sở hữu.

- **Không join cross-service**  
  Tránh join trực tiếp giữa các service. Thay vào đó, nhân bản dữ liệu chỉ-đọc qua event (CDC/outbox) để phục vụ truy vấn cục bộ.

- **Chấp nhận eventual consistency**  
  Không cố gắng đồng bộ ngay lập tức giữa các service. Sử dụng Saga hoặc Process Manager để điều phối các quy trình nhiều bước.

- **Giao tiếp bất đồng bộ là mặc định**  
  Sự cố mạng là bình thường. Thiết kế idempotency, retry với backoff, sử dụng dead-letter queue.

- **Thay đổi schema là vĩnh viễn**  
  Luôn version hóa schema & event. Thay đổi theo quy trình expand → migrate → contract để đảm bảo backward compatibility.

- **Quan trắc & khả năng khôi phục quan trọng hơn hiệu năng tức thì**  
  Log có traceId, metrics, alerting, backup & point-in-time recovery phải được thiết lập trước khi go-live.

- **Tách biệt OLTP và OLAP**  
  Báo cáo/analytics đi qua read models hoặc data warehouse (CDC), không truy vấn trực tiếp OLTP của service.

---

## Checklist thực hành

### Thiết kế biên giới dữ liệu
- Vẽ bounded context, xác định aggregate là truth source.
- ID: dùng UUID/ULID, không lộ auto-increment qua ranh giới.
- Thời gian: lưu UTC, có created_at, updated_at, version (optimistic locking).
- Soft delete nếu cần khôi phục/audit, hoặc tách bảng *_audit.

### Giao tiếp & đồng bộ
- Outbox pattern: ghi sự kiện cùng transaction với thay đổi dữ liệu, worker đẩy ra broker.
  - Ví dụ bảng outbox:  
    `outbox(id, aggregate_id, type, payload_json, occurred_at, processed_at null)`
- Inbox/Idempotency ở consumer: lưu message_id, processed_at để chống xử lý trùng.
- Dùng schema registry (Avro/Protobuf/JSON-Schema) + event versioning (chỉ thêm field, không đổi nghĩa cũ).

### Giao dịch phân tán
- Saga (choreography/orchestration): mỗi bước có compensation.
- Tránh 2PC. Ghi rõ state machine của quy trình dài (pending → confirmed → failed…).

### Truy vấn & hiệu năng
- CQRS nhẹ: write model tối ưu ghi, read model tối ưu đọc/denormalize.
- Index theo access pattern thực tế; đọc nhiều thì tách read replica.
- Caching: cache-aside, phải có TTL & chiến lược invalidation (ưu tiên event-driven).
- Phân trang: dùng keyset/created_at + id thay vì offset lớn.

### Di trú & phát hành
- Migration không phá vỡ:
  - Thêm cột/bảng mới (read-path ẩn)
  - Code ghi song song (dual-write an toàn bằng outbox)
  - Backfill
  - Chuyển read-path
  - Xoá phần cũ
- Tự động hóa migration và roll-back plan (đặc biệt khi thay đổi dữ liệu).

### An toàn & tuân thủ
- Least privilege: mỗi service có tài khoản DB riêng, chỉ cấp quyền cần thiết.
- Lưu secrets trong vault, không commit vào code.
- Mask/Encrypt PII, audit log truy cập.
- Multi-tenant:  
  - Ít tenant: schema-per-tenant (cô lập tốt)  
  - Nhiều tenant: row-level (rẻ)  
  - Cô lập mạnh nhất: DB-per-tenant

### Vận hành & SRE
- Connection pooling (ví dụ: pgbouncer cho Postgres), tránh cạn kết nối khi burst.
- Observability: log có trace_id, metrics (latency, error rate, consumer lag), distributed tracing.
- Runbook: xử lý nghẽn queue, poison message, backfill event, khôi phục từ backup.
- Test: contract test giữa producer/consumer, fixture dữ liệu, test chaos (mất partition, chậm network…).

---

## Mantra & lưu ý thực chiến

- “Một service, một sự thật. Không mượn cơm nhà người khác.”
- “Không share database, share sự kiện.”
- “Exactly-once là ước mơ; at-least-once + idempotent là thực tế.”
- “Bạn không join qua mạng—bạn replicate.”
- “Schema thay đổi thì code hai bên đều đau—hãy version hoá.”
- “CAP không phải triết học—hãy chọn trade-off cho từng use case.”
- “Backup mà chưa thử restore thì coi như chưa có.”
- “Đừng tối ưu sớm—tối ưu đường chẩn đoán trước.”
- “Báo cáo là hệ thống khác. Đừng bòn rút OLTP.”
- “Khi nghi ngờ, ghi thêm event. Lịch sử cứu bạn.”

---

## Khung xương tối thiểu cho một service

- Bảng domain có id (ULID), version, created_at, updated_at.
- Outbox + worker đẩy event.
- Consumer có inbox(processed_message_id, processed_at).
- Config retry (exponential backoff), DLQ, metric cho consumer lag.
- Migration có script forward + rollback, CI chạy trước deploy.