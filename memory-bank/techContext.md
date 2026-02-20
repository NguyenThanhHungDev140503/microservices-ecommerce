# Tech Context

## Công nghệ sử dụng
- **Backend**: .NET Core 8, C#, Minimal APIs.
- **Frontend**: React, Vite, Redux Toolkit, Material UI.
- **Data Access**: EF Core, Dapper, Marten, MongoDB .NET Driver.
- **Messaging**: MassTransit, RabbitMQ.
- **Search**: Elasticsearch.
- **Identity**: Keycloak.
- **Containerization**: Docker, Docker Compose, Kubernetes.

## Thiết lập phát triển
- Hệ điều hành: Windows 11 (WSL2 - Ubuntu).
- IDE: Visual Studio Code / Visual Studio.
- Runtime: Docker for Desktop.

## Ràng buộc kỹ thuật
- Các service microservice giao tiếp bất đồng bộ qua RabbitMQ.
- Sử dụng .NET 8 làm standard cho backend.
- Đảm bảo tính nhất quán qua Outbox Pattern.
