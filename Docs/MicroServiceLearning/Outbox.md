# Outbox Pattern trong Microservices

Outbox Pattern là một giải pháp phổ biến giúp đảm bảo **tính nhất quán dữ liệu** (data consistency) khi một service vừa cần ghi dữ liệu vào database, vừa cần phát sự kiện (event) ra message broker (Kafka, RabbitMQ, v.v.).

## 1. Vấn đề thường gặp

Trong hệ thống microservices, một service thường phải thực hiện hai bước:
- Ghi dữ liệu vào database (transactional DB)
- Gửi event sang message broker để các service khác xử lý tiếp

Nếu hai bước này **không được thực hiện atomically** (trong cùng một transaction), có thể xảy ra các lỗi:
- Ghi DB thành công nhưng gửi event thất bại → service khác không biết có thay đổi
- Gửi event thành công nhưng DB rollback → phát event “ma” (không đúng thực tế)

Điều này dẫn đến **mất đồng bộ** giữa database và event stream.

## 2. Giải pháp: Outbox Pattern

**Outbox Pattern** giải quyết vấn đề trên bằng cách:

- **Ghi dữ liệu chính và event vào cùng một transaction** trong database (thường là một bảng `outbox`)
- Một process/worker riêng biệt (gọi là **Outbox Relay**) sẽ đọc các event từ bảng outbox và gửi sang message broker
- Khi gửi thành công, event trong bảng outbox sẽ được đánh dấu là đã xử lý (`processed`)

**Lợi ích:** Event luôn đồng bộ với DB vì chúng được commit trong cùng transaction.

## 3. Kiến trúc tổng quan