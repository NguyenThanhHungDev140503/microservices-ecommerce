# System Patterns

## Kiến trúc hệ thống
Hệ thống sử dụng kiến trúc Microservices kết hợp với:
- **Clean Architecture**: Tách biệt logic nghiệp vụ khỏi infrastructure.
- **Vertical Slice Architecture**: Được áp dụng trong một số service để giảm bớt sự phức tạp của việc phân lớp.
- **CQRS**: Tách biệt luồng đọc và ghi (sử dụng MediatR).
- **Event-Driven Architecture**: Giao tiếp giữa các service thông qua Integration Events và RabbitMQ.

## Các mẫu thiết kế chính
- **Repository & Unit of Work**: Trừu tượng hóa truy cập dữ liệu.
- **Outbox Pattern**: Đảm bảo phân phối sự kiện đáng tin cậy.
- **Saga Pattern**: Quản lý giao dịch phân tán (Choreography-based).
- **API Gateway (YARP)**: Định tuyến yêu cầu và xử lý các vấn đề xuyên suốt (cross-cutting concerns).

## Thành phần hạ tầng
- **Databases**: PostgreSQL, MySQL, SQL Server, MongoDB, Redis.
- **Messaging**: RabbitMQ (MassTransit).
- **Security**: Keycloak (OIDC).
- **Observability**: Prometheus, Grafana, Loki, Tempo, OpenTelemetry.
