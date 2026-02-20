# Product Context

## Tại sao dự án này tồn tại?
Dự án được tạo ra để minh họa cách xây dựng một hệ thống Microservices thực tế với .NET Core, giải quyết các vấn đề về quy mô, khả năng bảo trì và tính nhất quán dữ liệu trong môi trường phân tán.

## Vấn đề cần giải quyết
- **Xử lý sự cố đồng bộ**: Sử dụng Event-Driven Architecture với RabbitMQ để giao tiếp bất đồng bộ.
- **Tính nhất quán dữ liệu**: Áp dụng Saga pattern và Outbox/Inbox pattern.
- **Quản lý phức tạp**: Phân rã hệ thống thành các dịch vụ nhỏ (Bounded Contexts) theo DDD.
- **Khả năng quan sát (Observability)**: Tích hợp OpenTelemetry, Prometheus, Grafana, Loki, Tempo.

## Trải nghiệm người dùng mục tiêu
- Người dùng cuối có thể tìm kiếm, thêm vào giỏ hàng và đặt hàng một cách mượt mà.
- Admin có thể quản lý catalog, đơn hàng, kho hàng và xem báo cáo.
