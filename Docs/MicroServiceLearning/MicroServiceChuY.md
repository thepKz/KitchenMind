Nguyên tắc cốt lõi (bạn nên thuộc lòng)

Một service = một database (ownership rõ ràng).
Không share DB giữa các service. Mọi truy cập dữ liệu phải đi qua API/Event của service chủ sở hữu.

Tránh join cross-service.
Thay vì join trực tiếp, nhân bản dữ liệu chỉ-đọc bằng event (CDC/outbox) để phục vụ query cục bộ.

Chấp nhận eventual consistency.
Đừng cố đồng bộ “ngay tức thì” giữa các service. Dùng Saga/Process Manager để điều phối quy trình nhiều bước.

Giao tiếp bất đồng bộ là mặc định.
Sự cố mạng là bình thường: thiết kế idempotency, retry với backoff, dead-letter queue.

Schema change là vĩnh viễn.
Version hoá schema & event; thay đổi theo kiểu expand → migrate → contract (backward compatible).

Quan trắc & khả năng khôi phục > hiệu năng tức thì.
Log có tương quan traceId, metrics, alerting, backup & point-in-time recovery trước khi go-live.

Tách OLTP và OLAP.
Báo cáo/analytics đi qua read models / data warehouse (CDC), không query trực tiếp OLTP của service.

Checklist thực hành (ngắn gọn mà “đủ đô”)
Thiết kế biên giới dữ liệu

Vẽ bounded contexts; chỉ ra aggregate nào là truth source.

Quy ước ID: UUID/ULID (sắp xếp tốt), không lộ auto-increment qua ranh giới.

Time: lưu UTC, có created_at, updated_at, version (optimistic locking).

Soft delete nếu có nhu cầu khôi phục/audit; hoặc tách bảng *_audit.

Giao tiếp & đồng bộ

Outbox pattern: ghi sự kiện cùng transaction với thay đổi dữ liệu → worker đẩy ra broker.
Ví dụ bảng outbox:

outbox(id, aggregate_id, type, payload_json, occurred_at, processed_at null)


Inbox/Idempotency ở consumer: lưu message_id, processed_at để chống xử lý trùng.

Schema registry (Avro/Protobuf/JSON-Schema) + event versioning (thêm field, không đổi nghĩa cũ).

Giao dịch phân tán

Saga (choreography/orchestration): mỗi bước có compensation.

Tránh 2PC. Ghi rõ state machine của quy trình dài (pending → confirmed → failed…).

Truy vấn & hiệu năng

CQRS nhẹ: write model tối ưu ghi; read model tối ưu đọc/denormalize.

Index theo access pattern thực tế; đọc nhiều thì tách read replica.

Caching: cache-aside; phải có TTL & chiến lược invalidation (event-driven càng tốt).

Phân trang: keyset/created_at + id thay vì offset lớn.

Di trú & phát hành

Migration không phá vỡ:

Thêm cột/bảng mới (read-path ẩn),

Code ghi song song (dual-write an toàn bằng outbox),

Backfill,

Chuyển read-path,

Xoá phần cũ.

Tự động hoá migration và roll-back plan (đặc biệt khi thay đổi dữ liệu).

An toàn & tuân thủ

Least privilege: tài khoản DB per service, chỉ quyền cần thiết.

Secrets trong vault, không commit.

Mask/Encrypt PII, audit log truy cập.

Multi-tenant: schema-per-tenant (ít tenant, cô lập tốt), row-level (nhiều tenant, rẻ), hoặc DB-per-tenant (cô lập mạnh nhất).

Vận hành & SRE

Connection pooling (PG: pgbouncer), tránh cạn kết nối khi burst.

Observability: log có trace_id, metrics (latency, error rate, consumer lag), distributed tracing.

Runbooks: nghẽn queue, poison message, backfill event, khôi phục từ backup.

Test: contract test giữa producer/consumer, fixture dữ liệu, test chaos (mất 1 partition, chậm network…).

Những “câu nói của tiền bối” (mantra đáng nhớ)

“Một service, một sự thật. Không mượn cơm nhà người khác.”

“Không share database, share sự kiện.”

“Exactly-once là ước mơ; at-least-once + idempotent là thực tế.”

“Bạn không join qua mạng—bạn replicate.”

“Schema thay đổi thì code hai bên đều đau—hãy version hoá.”

“CAP không phải triết học—hãy chọn trade-off cho từng use case.”

“Backup mà chưa thử restore thì coi như chưa có.”

“Đừng tối ưu sớm—tối ưu đường chẩn đoán trước.”

“Báo cáo là hệ thống khác. Đừng bòn rút OLTP.”

“Khi nghi ngờ, ghi thêm event. Lịch sử cứu bạn.”

Mẫu “khung xương” tối thiểu cho một service (gợi ý)

Bảng domain có id (ULID), version, created_at, updated_at.

outbox + worker đẩy event.

Consumer có inbox(processed_message_id, processed_at).

Config retry (exponential backoff), DLQ, metric cho consumer lag.

Migration có script forward + rollback, CI chạy trước deploy.