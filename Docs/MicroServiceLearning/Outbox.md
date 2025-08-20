Outbox Pattern trong microservices là một giải pháp đảm bảo tính nhất quán dữ liệu (data consistency) khi bạn vừa cần ghi dữ liệu vào database, vừa cần phát sự kiện (event) ra message broker (Kafka, RabbitMQ, v.v.).

Vấn đề

Trong hệ thống microservices, một service thường:

Lưu dữ liệu vào database (transactional DB).

Gửi event sang message broker để các service khác xử lý tiếp.

Nếu 2 bước này không được xử lý atomically thì có thể gặp lỗi:

Lưu DB thành công nhưng gửi event thất bại → service khác không biết có thay đổi.

Gửi event thành công nhưng DB rollback → phát event “ma”.

Điều này tạo ra tình trạng mất đồng bộ giữa DB và event stream.

Giải pháp Outbox Pattern

Thay vì ghi trực tiếp vào message broker, ta:

Transaction duy nhất: Ghi dữ liệu chính và event vào cùng database (table “outbox”).

Một process/worker riêng (Outbox Relay) đọc từ bảng outbox, gửi event sang message broker.

Khi gửi thành công, record trong outbox được đánh dấu “processed”.

👉 Nhờ vậy, event luôn đồng bộ với DB vì chúng được commit trong cùng transaction.

Kiến trúc tổng quan
Service A
 ├── Transaction DB
 │     ├── business_table (order, user, v.v.)
 │     └── outbox_table (pending events)
 └── Outbox Relay
        ├── đọc outbox_table
        ├── gửi message -> Kafka/RabbitMQ
        └── đánh dấu đã gửi

Ưu điểm

Đảm bảo atomicity giữa DB và event (không lo mất đồng bộ).

Không phụ thuộc vào message broker hỗ trợ XA transaction.

Có thể retry an toàn nếu gửi event thất bại.

Nhược điểm

Cần thêm cơ chế dọn dẹp (cleanup) bảng outbox.

Độ trễ cao hơn chút (event không phát ngay lập tức, mà qua relay).

Tăng thêm phức tạp trong hệ thống.

So sánh với Transactional Outbox + CDC

Transactional Outbox (classic): app ghi vào outbox, relay đọc và gửi đi.

CDC (Change Data Capture): dùng Debezium hoặc binlog listener để phát hiện thay đổi ở outbox table và đẩy sự kiện đi → giảm code nhưng cần hạ tầng phức tạp hơn.

👉 Tóm lại: Outbox Pattern là cách phổ biến để giải quyết bài toán "làm sao phát event mà vẫn đảm bảo dữ liệu và event luôn đồng bộ" trong microservices.

Bạn có muốn mình vẽ sequence diagram minh họa dòng chảy (DB + Outbox + Message Broker) cho dễ hình dung không?