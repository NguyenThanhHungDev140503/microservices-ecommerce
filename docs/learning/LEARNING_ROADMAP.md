# 🚀 Lộ Trình Học Tập .NET Microservices
## Dự án: Progcoder Shop Microservices

> **Mục tiêu:** Từ người mới hiểu .NET và REST API → Trở thành Developer thành thạo kiến trúc Microservices
>
> **Thời gian dự kiến:** 6-8 tuần (2-3 tháng với tốc độ vừa phải)
>
> **Cách sử dụng:** Bám theo từng Phase, hoàn thành Task, đánh dấu [✅] khi hoàn thành

---

## 📋 Tổng Quan Kiến Trúc Dự Án

### Các Service chính (7 services)
```
├── Catalog Service (Quản lý sản phẩm)
├── Basket Service (Giỏ hàng - dùng Redis)
├── Order Service (Đơn hàng - phức tạp nhất)
├── Inventory Service (Kho hàng)
├── Discount Service (Giảm giá)
├── Notification Service (Thông báo)
└── Report Service (Báo cáo - dùng gRPC)
```

### Các công nghệ bạn sẽ học
| Cấp độ | Công nghệ | Mục đích |
|--------|-----------|----------|
| Cơ bản | Clean Architecture, DDD, MediatR | Tổ chức code chuẩn |
| Trung cấp | gRPC, RabbitMQ + MassTransit, YARP Gateway | Giao tiếp giữa services |
| Nâng cao | Outbox Pattern, Redis, Polly, Docker | Độ tin cậy và hiệu năng |

---

## Phase 1: Nền Tảng Của Một Service (2-3 tuần)

### 🎯 Mục tiêu
- Hiểu và áp dụng Clean Architecture
- Nắm vững DDD (Domain-Driven Design)
- Thực hành CQRS với MediatR
- Hiểu cách cấu trúc một project chuẩn .NET

### 📚 Thứ tự học

#### Week 1: Clean Architecture & DDD

**Day 1-2: Hiểu cấu trúc thư mục**
- **Task 1.1:** [ ] Mở `src/Services/Order` và vẽ sơ đồ thư mục ra giấy
  ```
  Order/
  ├── Core/
  │   ├── Order.Domain/       ← Trái tim nghiệp vụ (Entities, Value Objects)
  │   ├── Order.Application/  ← Use Cases (Commands, Queries)
  │   └── Order.Infrastructure/ ← Kết nối DB (EF Core)
  ├── Api/
  │   ├── Order.Api/          ← REST API Controllers
  │   └── Order.Grpc/         ← gRPC Services
  └── Worker/
      ├── Order.Worker.Consumer/    ← Nhận messages từ RabbitMQ
      └── Order.Worker.Outbox/      ← Gửi messages từ Database ra RabbitMQ
  ```

- **Task 1.2:** [ ] Tìm hiểu Dependency Inversion Principle (DIP)
  - **File xem:** `src/Services/Order/Core/Order.Domain/Entities/Order.cs`
  - **Câu hỏi:** Tại sao Entities không phụ thuộc vào EF Core hoặc bất kỳ thư viện nào?

- **Task 1.3:** [ ] Hiểu Value Objects vs Entities
  - **File xem:** `src/Services/Order/Core/Order.Domain/ValueObjects/` (nếu có)
  - **Bài tập:** Tìm ra các trường nào nên là Value Object (ví dụ: Money, Address)

**Day 3-4: Domain Layer (Quan trọng nhất!)**

- **Task 1.4:** [ ] Nghiên cứu Entities trong Order Service
  - **File xem:** `src/Services/Order/Core/Order.Domain/Entities/`
  - **Đọc:** `Order.cs`, `OrderItem.cs`
  - **Câu hỏi:** Các quy tắc nghiệp vụ (business rules) nằm ở đâu?
    - Ví dụ: "Order không thể có status là 'Completed' nếu chưa thanh toán"

- **Task 1.5:** [ ] Tìm hiểu Aggregates & Aggregate Root
  - **Bài tập:** Trong Order Service, đâu là Aggregate Root?
    - Gợi ý: Order hay OrderItem? Tại sao?

- **Task 1.6:** [ ] Tìm hiểu Repository Pattern
  - **File xem:** `src/Services/Order/Core/Order.Domain/Interfaces/IOrderRepository.cs`
  - **Câu hỏi:** Tại sao cần Repository thay vì dùng DbContext trực tiếp?

**Day 5-7: Application Layer**

- **Task 1.7:** [ ] Hiểu CQRS (Command Query Responsibility Segregation)
  - **File xem:** `src/Services/Order/Core/Order.Application/`
  - **Đọc:**
    - `Commands/CreateOrderCommand.cs`
    - `Queries/GetOrderByIdQuery.cs`
    - `Handlers/CreateOrderCommandHandler.cs`
  - **Câu hỏi:** Tại sao phải tách Command và Query?

- **Task 1.8:** [ ] Nghiên cứu MediatR
  - **File xem:** `src/Services/Order/Core/Order.Application/Handlers/`
  - **Tìm kiếm:** Tất cả các class implement `IRequestHandler<,>`
  - **Bài tập:** Vẽ flow khi client gọi `POST /api/orders`:
    ```
    Controller → MediatR → Handler → Repository → Database
    ```

**Assignment Week 1:**
- [ ] Tạo một file README.md giải thích cấu trúc Order Service
- [ ] Vẽ sơ đồ class diagram cho Order Entity

---

#### Week 2: Validation, Infrastructure & Testing

**Day 8-9: Validation Layer**

- **Task 2.1:** [ ] Học FluentValidation
  - **File xem:** `src/Services/Order/Core/Order.Application/Validators/`
  - **Đọc:** `CreateOrderCommandValidator.cs`
  - **Câu hỏi:** Tại sao không dùng `[Required]` attribute của DataAnnotations?

- **Task 2.2:** [ ] Hiểu Pipeline Behavior (MediatR)
  - **File xem:** `src/Services/Order/Core/Order.Application/Behaviors/`
  - **Đọc:** `ValidationBehavior.cs`
  - **Bài tập:** Vẽ pipeline khi MediatR nhận request:
    ```
    ValidationBehavior → LoggingBehavior → Handler
    ```

**Day 10-12: Infrastructure Layer**

- **Task 2.3:** [ ] Nghiên cứu Entity Framework Core
  - **File xem:** `src/Services/Order/Core/Order.Infrastructure/`
  - **Đọc:**
    - `Data/OrderDbContext.cs`
    - `Configurations/OrderConfiguration.cs` (Fluent API)
  - **Bài tập:**
    - Tìm hiểu cách EF Core map Entities vào Database Tables
    - Tìm hiểu Indexes (Database Index) trong Config

- **Task 2.4:** [ ] Hiểu Repository Implementation
  - **File xem:** `src/Services/Order/Core/Order.Infrastructure/Repositories/`
  - **Đọc:** `OrderRepository.cs`
  - **Câu hỏi:** Tại sao Repository không chứa business logic?

**Day 13-14: API Layer & Dependency Injection**

- **Task 2.5:** [ ] Nghiên cứu Minimal APIs hoặc Controllers
  - **File xem:** `src/Services/Order/Api/Order.Api/`
  - **Đọc:** `Program.cs` hoặc `Controllers/`
  - **Câu hỏi:** Tại sao API layer chỉ nên có code "thin" (mỏng)?

- **Task 2.6:** [ ] Hiểu Dependency Injection (DI)
  - **File xem:** `src/Services/Order/Api/Order.Api/Program.cs`
  - **Bài tập:**
    - Tìm tất cả `services.AddScoped`, `services.AddTransient`, `services.AddSingleton`
    - Định nghĩa sự khác nhau giữa 3 kiểu lifetime này
    - Vẽ diagram lifecycle của các service

**Assignment Week 2:**
- [ ] Thêm một validation rule mới cho CreateOrderCommand
- [ ] Implement một Repository method mới (ví dụ: GetOrdersByDate)
- [ ] Test API bằng Postman hoặc Swagger UI

---

## Phase 2: Giao Thép Giữa Services (2-3 tuần)

### 🎯 Mục tiêu
- Thực hành gRPC cho giao tiếp đồng bộ
- Hiểu RabbitMQ và Message-driven architecture
- Áp dụng MassTransit để implement Event-driven
- Nắm vững API Gateway pattern

### 📚 Thứ tự học

#### Week 3: gRPC - Giao tiếp đồng bộ

**Day 15-16: Cơ bản về gRPC**

- **Task 3.1:** [ ] Hiểu khái niệm gRPC vs REST API
  - **File xem:** `src/Services/Order/Api/Order.Grpc/`
  - **Đọc:**
    - `Protos/order.proto`
    - `Services/OrderGrpcService.cs`
  - **Câu hỏi:**
    - gRPC dùng protocol gì? (HTTP/1.1 hay HTTP/2?)
    - Tại sao gRPC nhanh hơn REST?

- **Task 3.2:** [ ] Tìm hiểu Protocol Buffers (Protobuf)
  - **File xem:** Các file `.proto` trong các service
  - **Bài tập:**
    - Định nghĩa message trong `.proto`
    - So sánh Protobuf vs JSON

**Day 17-18: gRPC Client (Client calls gRPC)**

- **Task 3.3:** [ ] Nghiên cứu cách gọi gRPC từ một service khác
  - **Ví dụ:** Inventory Service gọi Order Service
  - **File xem:** `src/Services/Inventory/Core/Inventory.Infrastructure/Grpc/` (nếu có)
  - **Câu hỏi:** Làm sao để register gRPC client trong DI?

- **Task 3.4:** [ ] Bài tập thực hành:
  - Tạo một gRPC service mới (ví dụ: GetProductDetails)
  - Implement logic trả về thông tin sản phẩm
  - Test bằng gRPC client tool (ví dụ: grpcurl)

**Day 19-21: gRPC Best Practices**

- **Task 3.5:** [ ] Hiểu Interceptors (Middleware của gRPC)
  - **Bài tập:** Tìm hiểu cách add logging interceptor cho tất cả gRPC calls

- **Task 3.6:** [ ] Error Handling trong gRPC
  - **Câu hỏi:** Làm sao để throw exception từ gRPC service?
  - **Tìm kiếm:** Các status codes của gRPC (NotFound, InvalidArgument, etc.)

**Assignment Week 3:**
- [ ] Tạo một gRPC service hoàn chỉnh (Proto file + Implementation)
- [ ] Test gRPC service bằng Postman hoặc grpcurl
- [ ] So sánh performance giữa REST API và gRPC (sử dụng Apache Benchmark hoặc Postman collections)

---

#### Week 4: RabbitMQ & MassTransit - Giao tiếp bất đồng bộ

**Day 22-23: Cơ bản về Message Broker**

- **Task 4.1:** [ ] Hiểu khái niệm Message Queue
  - **File xem:** `src/Shared/EventSourcing/MassTransit/Extensions.cs`
  - **Đọc:** Cấu hình MassTransit với RabbitMQ
  - **Câu hỏi:**
    - Tại sao cần Message Broker thay vì REST API?
    - Khi nào dùng Sync (REST/gRPC), khi nào dùng Async (Message Queue)?

- **Task 4.2:** [ ] Khám phá RabbitMQ Management UI
  - **Task:** Mở `http://localhost:15672` (chạy `docker-compose up` trước)
  - **Bài tập:**
    - Xem các Queues, Exchanges, Bindings
    - Gửi message thủ công để test

**Day 24-25: MassTransit - Publishing Events**

- **Task 4.3:** [ ] Tìm hiểu cách Publish events
  - **File xem:** Tìm code dùng `IPublishEndpoint`
  - **Ví dụ:** `OrderCreatedIntegrationEvent`
  - **Bài tập:** Vẽ flow publish event:
    ```
    Service A → Publish Event → RabbitMQ → Service B consume
    ```

- **Task 4.4:** [ ] Định nghĩa Events
  - **File xem:** `src/Shared/Contracts/`
  - **Đọc:** Các file trong `Order.Contract/`, `Catalog.Contract/`
  - **Câu hỏi:** Tại sao Contracts phải nằm trong Shared project?

**Day 26-28: MassTransit - Consuming Events**

- **Task 4.5:** [ ] Nghiên cứu Consumers
  - **File xem:** `src/Services/Order/Worker/Order.Worker.Consumer/`
  - **Đọc:** Các class implement `IConsumer<T>`
  - **Bài tập:**
    - Vẽ flow khi một Event được published
    - Hiểu mechanism: MassTransit tự động scan và register Consumers

- **Task 4.6:** [ ] Retry Policies của MassTransit
  - **File xem:** Tìm cấu hình retry trong `Extensions.cs`
  - **Câu hỏi:**
    - Tại sao cần retry?
    - Làm thế nào để configure số lần retry?
    - Exponential backoff là gì?

- **Task 4.7:** [ ] Saga Pattern (nếu có trong dự án)
  - **File xem:** Tìm State Machine của MassTransit (nếu dùng)
  - **Bài tập:** Vẽ saga flow cho Order process:
    ```
    CreateOrder → ReserveInventory → ProcessPayment → ConfirmOrder
    ```

**Assignment Week 4:**
- [ ] Tạo một Integration Event mới (ví dụ: OrderCancelled)
- [ ] Implement Consumer cho Event đó
- [ ] Test bằng cách publish event thủ công qua RabbitMQ UI
- [ ] Đọc tài liệu: `docs/RabbitMQ/RabbitMQ_MassTransit_Onboarding_Guide_v2.md`

---

#### Week 5: API Gateway & Routing

**Day 29-30: Cơ bản về API Gateway**

- **Task 5.1:** [ ] Hiểu khái niệm API Gateway
  - **File xem:** `src/ApiGateway/YarpApiGateway/`
  - **Đọc:**
    - `Program.cs`
    - `appsettings.json` hoặc `appsettings.Development.json`
  - **Câu hỏi:**
    - Tại sao không cho frontend gọi trực tiếp đến các microservices?
    - Gateway đóng vai trò gì? (Routing, Auth, Rate Limiting, etc.)

- **Task 5.2:** [ ] Nghiên cứu YARP (Yet Another Reverse Proxy)
  - **Bài tập:**
    - Tìm hiểu cách YARP map incoming request → destination service
    - Đọc config trong `appsettings.json`:
      ```json
      "Routes": {
        "OrderRoute": {
          "ClusterId": "OrderCluster",
          "Match": { "Path": "/api/orders/{**catch-all}" }
        }
      },
      "Clusters": {
        "OrderCluster": { "Destinations": { "OrderApi": { "Address": "http://localhost:5001" } } }
      }
      ```

**Day 31-32: Advanced Gateway Features**

- **Task 5.3:** [ ] YARP Middleware & Transforms
  - **File xem:** `Program.cs` trong Gateway
  - **Bài tập:** Tìm hiểu Transforms (modify request/response headers, path, etc.)

- **Task 5.4:** [ ] Load Balancing
  - **Bài tập:** Hiểu cách Gateway phân phối request khi có nhiều instances của một service

**Day 33-35: Testing & Debugging Gateway**

- **Task 5.5:** [ ] Test Gateway routing
  - **Bài tập:**
    - Gọi `http://localhost:5000/api/orders` (Gateway) thay vì `http://localhost:5001` (Order Service)
    - Xem logs để xác minh request được forwarded

- **Task 5.6:** [ ] Troubleshooting Gateway issues
  - **Bài tập:**
    - Tạo một route sai → xem error message
    - Xem logs của Gateway để debug

**Assignment Week 5:**
- [ ] Thêm một route mới cho Catalog Service vào Gateway
- [ ] Implement header transformation (ví dụ: add `X-Request-ID` header)
- [ ] Test load balancing bằng cách chạy 2 instances của một service
- [ ] Đọc tài liệu về YARP: `https://microsoft.github.io/reverse-proxy/`

---

## Phase 3: Độ Tin Cậy & Hiệu Năng (2-3 tuần)

### 🎯 Mục tiêu
- Master Transactional Outbox Pattern
- Thực hành Distributed Caching với Redis
- Áp dụng Resilience Patterns với Polly
- Hiểu Distributed Transactions & Consistency

### 📚 Thứ tự học

#### Week 6: Outbox Pattern - Đảm bảo tính toàn vẹn dữ liệu

**Day 36-37: Vấn đề của Distributed Transactions**

- **Task 6.1:** [ ] Hiểu vấn đề: "What happens if..."
  - **Câu hỏi:**
    - Lưu Order vào Database thành công → Publish Event thất bại → Kết quả?
    - Publish Event thành công → Lưu Database thất bại → Kết quả?
  - **Bài tập:** Tìm hiểu CAP Theorem:
    - Consistency (C)
    - Availability (A)
    - Partition Tolerance (P)

- **Task 6.2:** [ ] Tìm hiểu 2PC (Two-Phase Commit)
  - **Câu hỏi:** Tại sao 2PC không phù hợp với microservices?

**Day 38-40: Outbox Pattern Implementation**

- **Task 6.3:** [ ] Nghiên cứu Outbox Pattern
  - **File xem:** `src/Services/Order/Worker/Order.Worker.Outbox/`
  - **Đọc:**
    - `Services/OutboxProcessor.cs` hoặc tương tự
    - `Entities/OutboxMessage.cs` trong Domain
  - **Bài tập:** Vẽ flow Outbox:
    ```
    1. Save Order + OutboxMessage (same transaction)
    2. OutboxProcessor reads OutboxMessage
    3. Publish Event to RabbitMQ
    4. Mark OutboxMessage as Processed
    ```

- **Task 6.4:** [ ] Tìm hiểu Idempotency
  - **Câu hỏi:**
    - Tại sao OutboxProcessor có thể process cùng message 2 lần?
    - Làm sao để đảm bảo idempotency (kết quả như nhau dù chạy bao nhiêu lần)?

- **Task 6.5:** [ ] Concurrency & Locking
  - **Bài tập:** Tìm hiểu cách OutboxProcessor tránh duplicate processing:
    - Database lock
    - Select-for-update
    - Message processing status (Pending → Processing → Processed/Failed)

**Assignment Week 6:**
- [ ] Tạo một diagram giải thích Outbox Pattern
- [ ] Thêm một integration event mới vào Outbox
- [ ] Test bằng cách dừng OutboxProcessor, tạo order, rồi chạy lại processor
- [ ] Đọc tài liệu: `docs/RabbitMQ/RabbitMQ_MassTransit_Architecture.md` (phần Outbox)

---

#### Week 7: Redis - Distributed Caching

**Day 41-42: Cơ bản về Redis**

- **Task 7.1:** [ ] Hiểu khái niệm In-Memory Database
  - **Câu hỏi:**
    - Tại sao Redis nhanh hơn SQL Server?
    - Redis lưu data ở đâu? (Disk hay RAM?)
    - Khi nào nên dùng Redis? (Cache, Session, Pub/Sub, etc.)

- **Task 7.2:** [ ] Redis Data Structures
  - **Bài tập:** Khám phá các data types của Redis:
    - String
    - Hash
    - List
    - Set
    - Sorted Set
  - **Câu hỏi:** Basket Service dùng data type nào?

**Day 43-44: Redis trong .NET**

- **Task 7.3:** [ ] Nghiên cứu cách dùng Redis trong .NET
  - **File xem:** `src/Services/Basket/Core/Basket.Infrastructure/`
  - **Đọc:**
    - `Repositories/BasketRepository.cs`
    - `Caching/` (nếu có)
  - **Bài tập:** Tìm hiểu:
    - Connection Multiplexer
    - GET/SET commands
    - Expiration (TTL - Time To Live)

- **Task 7.4:** [ ] Cache-Aside Pattern
  - **Bài tập:** Vẽ flow Cache-Aside:
    ```
    Request → Check Cache → Hit? Return
                          → Miss? Query DB → Update Cache → Return
    ```

**Day 45-47: Advanced Redis Patterns**

- **Task 7.5:** [ ] Cache Invalidation Strategies
  - **Câu hỏi:**
    - Khi cập nhật Catalog data → Làm sao để invalidate cache trong Basket?
    - Cache invalidation là một trong 2 vấn đề khó nhất của CS. Tại sao?

- **Task 7.6:** [ ] Distributed Lock
  - **Bài tập:** Tìm hiểu RedLock algorithm
  - **Câu hỏi:** Khi nào cần distributed lock? (Concurrent updates)

- **Task 7.7:** [ ] Redis Pub/Sub
  - **Bài tập:** Tìm hiểu cách dùng Redis Pub/Sub để invalidate cache
  - **Ví dụ:** Catalog Service publish event → Basket Service subscribe và clear cache

**Assignment Week 7:**
- [ ] Implement caching cho một endpoint (ví dụ: GetProducts)
- [ ] Test cache hit/miss bằng Redis CLI (`MONITOR` command)
- [ ] Implement distributed lock cho một operation (ví dụ: UpdateProduct)
- [ ] Đọc tài liệu về Redis patterns

---

#### Week 8: Resilience & Fault Tolerance

**Day 48-49: Cơ bản về Resilience**

- **Task 8.1:** [ ] Hiểu khái niệm Fault Tolerance
  - **Câu hỏi:**
    - Làm sao hệ thống vẫn hoạt động khi:
      - Database bị slow?
      - External API bị down?
      - Network bị lag?

- **Task 8.2:** [ ] Nghiên cứu Circuit Breaker Pattern
  - **Bài tập:** Vẽ diagram Circuit Breaker:
    ```
    Closed → Open → Half-Open → Closed/Failed
    ```
  - **Câu hỏi:**
    - Circuit Breaker khác với Retry thế nào?

**Day 50-51: Polly - Resilience Library cho .NET**

- **Task 8.3:** [ ] Cấu hình Polly trong .NET
  - **File xem:** Tìm kiếm `AddPolicyHandler` trong `Program.cs` hoặc `Extensions.cs`
  - **Bài tập:** Tìm hiểu:
    - Retry Policy (Thử lại N lần)
    - Circuit Breaker Policy
    - Timeout Policy
    - Fallback Policy

- **Task 8.4:** [ ] Implement Retry Policy
  - **Bài tập thực hành:**
    ```csharp
    services.AddHttpClient<ICatalogService, CatalogService>()
        .AddPolicyHandler(GetRetryPolicy());
    ```
  - **Câu hỏi:** Tại sao cần exponential backoff (thời gian retry tăng dần)?

**Day 52-54: Advanced Polly Patterns**

- **Task 8.5:** [ ] Circuit Breaker Implementation
  - **Bài tập:**
    - Configure Circuit Breaker: 3 failures → Open for 30 seconds
    - Test bằng cách stop service và xem Circuit Breaker hoạt động

- **Task 8.6:** [ ] Policy Composition (Kết hợp nhiều policies)
  - **Bài tập:**
    ```
    Retry Policy + Circuit Breaker + Timeout
    ```
  - **Câu hỏi:** Thứ tự apply policies quan trọng không?

- **Task 8.7:** [ ] Bulkhead Isolation
  - **Bài tập:** Tìm hiểu cách giới hạn số lượng concurrent requests
  - **Câu hỏi:** Khi nào cần Bulkhead? (Cascading failures)

**Assignment Week 8:**
- [ ] Implement Polly cho một HTTP client
- [ ] Test Circuit Breaker bằng cách shutdown service
- [ ] Monitor Polly logs (nhìn thấy retries, circuit state changes)
- [ ] Đọc tài liệu Polly: `https://github.com/App-vNext/Polly`

---

## Phase 4: DevOps & Hạ Tầng (1-2 tuần)

### 🎯 Mục tiêu
- Hiểu Docker & Containerization
- Docker Compose cho orchestration
- CI/CD Pipeline (nếu có)
- Monitoring & Logging

### 📚 Thứ tự học

#### Week 9: Docker & Containerization

**Day 55-56: Docker Fundamentals**

- **Task 9.1:** [ ] Hiểu khái niệm Container
  - **Câu hỏi:**
    - Container khác với VM thế nào?
    - Tại sao cần Docker?

- **Task 9.2:** [ ] Dockerfile Analysis
  - **File xem:** `Dockerfile` trong mỗi service
  - **Bài tập:** Đọc và hiểu từng dòng:
    ```dockerfile
    FROM mcr.microsoft.com/dotnet/aspnet:8.0
    WORKDIR /app
    COPY bin/Release/net8.0/publish .
    ENTRYPOINT ["dotnet", "Order.Api.dll"]
    ```

**Day 57-58: Docker Compose**

- **Task 9.3:** [ ] Nghiên cứu docker-compose.yml
  - **File xem:** `docker-compose.yml` ở thư mục gốc
  - **Bài tập:**
    - Hiểu structure: `services`, `networks`, `volumes`
    - Tìm hiểu depends_on và healthcheck
  - **Câu hỏi:**
    - Làm sao các services connect với nhau? (Container network)
    - Environment variables dùng để làm gì?

- **Task 9.4:** [ ] Test Docker Compose
  - **Bài tập:**
    ```bash
    docker-compose up -d
    docker-compose logs -f order.api
    docker-compose down
    ```

**Day 59-60: Advanced Docker Topics**

- **Task 9.5:** [ ] Multi-stage Build
  - **Bài tập:** So sánh image size khi dùng vs không dùng multi-stage build
  - **Câu hỏi:** Tại sao cần giảm image size?

- **Task 9.6:** [ ] Docker Volumes & Networks
  - **Bài tập:**
    - Hiểu cách persist data với volumes (Database, Redis)
    - Tìm hiểu bridge vs overlay networks

**Assignment Week 9:**
- [ ] Tạo Dockerfile cho một project nhỏ mới
- [ ] Build và chạy container bằng Docker CLI
- [ ] Thêm service mới vào docker-compose.yml
- [ ] Test toàn bộ stack bằng `docker-compose up`

---

#### Week 10: Monitoring, Logging & Observability

**Day 61-62: Logging Strategy**

- **Task 10.1:** [ ] Nghiên cứu Logging trong .NET
  - **Bài tập:**
    - Tìm hiểu ILogger vs Console.WriteLine
    - Structured Logging (Serilog, Seq)
  - **File xem:** Tìm cấu hình Serilog trong `appsettings.json`

- **Task 10.2:** [ ] Correlation ID / Request Tracing
  - **Bài tập:** Tìm hiểu cách trace request qua nhiều services
  - **Câu hỏi:** Làm sao để biết 1 request đã đi qua những services nào?

**Day 63-64: Application Performance Monitoring (APM)**

- **Task 10.3:** [ ] Nghiên cứu OpenTelemetry
  - **Bài tập:** Tìm hiểu Metrics, Traces, Logs (3 pillars of observability)
  - **File xem:** Tìm cấu hình OpenTelemetry trong `Program.cs`

- **Task 10.4:** [ ] Health Checks
  - **Bài tập:** Test endpoint `/health` và `/healthz`
  - **Câu hỏi:** Health checks dùng để làm gì? (Load balancer, orchestrator)

**Day 65-66: Distributed Tracing**

- **Task 10.5:** [ ] Tìm hiểu Distributed Tracing
  - **Bài tập:**
    - Vẽ diagram trace ID và span IDs
    - Tìm hiểu Jaeger / Zipkin / OpenTelemetry Collector

- **Task 10.6:** [ ] Debugging Production Issues
  - **Bài tập:** Tìm hiểu cách sử dụng logs và traces để debug:
    - Request timeout
    - Database slow query
    - Circuit breaker opened

**Assignment Week 10:**
- [ ] Thêm structured logging cho một service
- [ ] Implement health checks với dependency checks (Database, RabbitMQ)
- [ ] Visualize distributed tracing bằng Jaeger (nếu có setup)
- [ ] Đọc tài liệu về observability best practices

---

## 🎓 Phase 5: Nâng Cao & Chuyên Sâu (Tùy chọn)

### Topics for Advanced Learners

**Event Sourcing & CQRS (Advanced)**
- **Nghiên cứu:** `src/Shared/EventSourcing/`
- **File xem:** Event Store implementation (nếu có)
- **Bài tập:**
  - Tìm hiểu cách lưu events thay vì current state
  - Tìm hiểu Event Sourcing vs CRUD

**Saga Pattern (Orchestration vs Choreography)**
- **File xem:** Tìm State Machine trong MassTransit
- **Bài tập:**
  - Vẽ saga flow cho Order process với compensating transactions
  - Tìm hiểu MassTransit Courier

**Security & Authentication**
- **File xem:** Configuration về JWT, OAuth2 (nếu có)
- **Bài tập:**
  - Tìm hiểu cách implement JWT tokens
  - Tìm hiểu IdentityServer4 / Duende IdentityServer

**Testing Strategy**
- **Unit Tests:** xUnit, Moq, NSubstitute
- **Integration Tests:** WebApplicationFactory
- **Contract Tests:** Pact
- **Bài tập:**
  - Viết unit tests cho Handler
  - Viết integration tests cho API

**Performance Optimization**
- **Bài tập:**
  - Database indexing strategy
  - N+1 query problem
  - Caching strategies (Read-through, Write-through, Write-back)
  - Asynchronous processing (BackgroundService)

---

## 📊 Milestones & Checkpoints

### Milestone 1 (Sau Phase 1 - Cơ bản)
- [ ] Có thể giải thích cấu trúc Order Service
- [ ] Có thể tạo một new service với Clean Architecture
- [ ] Hiểu DDD: Entities, Value Objects, Aggregates
- [ ] Thực hành được MediatR CQRS

### Milestone 2 (Sau Phase 2 - Giao tiếp)
- [ ] Có thể tạo gRPC service và client
- [ ] Có thể publish/consume events với MassTransit
- [ ] Có thể cấu hình API Gateway với YARP
- [ ] Hiểu khi nào dùng sync, khi nào dùng async

### Milestone 3 (Sau Phase 3 - Độ tin cậy)
- [ ] Có thể giải thích và implement Outbox Pattern
- [ ] Có thể dùng Redis cho caching
- [ ] Có thể configure Polly cho resilience
- [ ] Hiểu distributed transactions & consistency

### Milestone 4 (Sau Phase 4 - DevOps)
- [ ] Có thể build và run Docker containers
- [ ] Có thể orchestrate với docker-compose
- [ ] Có thể debug production issues với logs & traces
- [ ] Có thể implement health checks

---

## 📚 Tài Liệu Tham Khảo Trong Project

### Các file documentation quan trọng
- `docs/RabbitMQ/RabbitMQ_MassTransit_Onboarding_Guide_v2.md` - Hướng dẫn MassTransit
- `docs/RabbitMQ/RabbitMQ_MassTransit_Architecture.md` - Kiến trúc Event-driven
- `docs/RabbitMQ/RabbitMQ_MassTransit_Usage_Guide.md` - Cách dùng RabbitMQ
- `docs/Messaging/Messaging_Reference.md` - Tổng quan Messaging
- `docs/Messaging/Messaging_Consumers_Reference.md` - Danh sách Consumers
- `docs/analysis/cache-deep-analysis.md` - Phân tích Cache

### External Resources
- [MassTransit Documentation](https://masstransit.io/)
- [gRPC Documentation](https://grpc.io/docs/languages/dotnet/)
- [YARP Documentation](https://microsoft.github.io/reverse-proxy/)
- [Polly Documentation](https://github.com/App-vNext/Polly)
- [Redis Documentation](https://redis.io/docs/)
- [Docker Documentation](https://docs.docker.com/)

---

## 🚀 Tips Học Tập Hiệu Quả

### 1. Learn by Doing - Học qua thực hành
- Đừng chỉ đọc code, hãy **modify code**
- Tạo branch mới cho mỗi task
- Commit với message mô tả task đã làm

### 2. Draw Diagrams - Vẽ sơ đồ
- Vẽ flow diagram cho mỗi feature
- Visualize architecture trên giấy hoặc dùng tool (draw.io, Miro)
- Update diagram khi hiểu rõ hơn

### 3. Ask "Why" - Hỏi "Tại sao"
- Đừng chỉ biết "HOW" (làm thế nào), hãy hiểu "WHY" (tại sao)
- Tại sao dùng pattern này? Tại sao không dùng cách khác?
- Trade-off (đổi chác) của mỗi giải pháp là gì?

### 4. Break Things - Phá vỡ để hiểu
- Stop Database service → xem system hoạt động thế nào
- Stop RabbitMQ → xem message bị đợi như thế nào
- Test Circuit Breaker bằng cách shutdown service
- Điều này giúp bạn hiểu sâu hơn

### 5. Read Tests - Đọc unit tests
- Tests là documentation tốt nhất
- Nó cho bạn biết code được expect hoạt động thế nào
- Nó cho bạn biết edge cases (trường hợp đặc biệt)

### 6. Code Review - Xem code người khác
- So sánh cách implement trong các service khác nhau
- Tìm hiểu best practices từ codebase
- Note: Có nhiều cách để implement một feature

### 7. Take Notes - Ghi chép
- Tạo repo riêng để lưu notes, diagrams
- Viết blogpost về những gì đã học
- Teaching là cách học tốt nhất

---

## 🎯 Final Project

### Assignment Tổng Hợp
Khi hoàn thành 4 Phase, hãy thử thực hiện một mini-project:

**Project: "Microservices Coffee Shop"**
- [ ] Coffee Catalog Service (CRUD menu)
- [ ] Order Service (CreateOrder, gRPC)
- [ ] Inventory Service (Check availability, RabbitMQ events)
- [ ] Payment Service (Process payment, Outbox Pattern)
- [ ] Notification Service (Send email, Redis caching)
- [ ] API Gateway (YARP routing)
- [ ] Docker Compose (orchestrate all services)
- [ ] Polly resilience (retry, circuit breaker)
- [ ] Monitoring (logs, health checks)

### Success Criteria
- [ ] Tất cả services chạy với Docker Compose
- [ ] Có thể tạo order end-to-end (client → Gateway → Order → Inventory → Payment → Notification)
- [ ] Có resilience: Database downtime không crash system
- [ ] Có logging: Debug production issue dễ dàng
- [ ] Có documentation: README.md cho mỗi service

---

## 📅 Timeline Gợi Ý

| Phase | Time | Focus |
|-------|------|-------|
| Phase 1 | 2-3 weeks | Clean Architecture, DDD, CQRS |
| Phase 2 | 2-3 weeks | gRPC, RabbitMQ, MassTransit, Gateway |
| Phase 3 | 2-3 weeks | Outbox, Redis, Polly |
| Phase 4 | 1-2 weeks | Docker, Monitoring |
| Phase 5 | Tùy chọn | Event Sourcing, Advanced patterns |
| **Tổng** | **7-11 weeks** | **Full-stack Microservices** |

---

## 🤝 Cần Hỗ Trợ?

Nếu gặp khó khăn ở đâu:
1. **Debug logs:** Chạy `docker-compose logs -f <service-name>`
2. **RabbitMQ UI:** `http://localhost:15672`
3. **Swagger UI:** `http://localhost:<port>/swagger`
4. **Database:** Kết nối với SQL Server từ SSMS hoặc Azure Data Studio

---

**Chúc bạn học tập hiệu quả và trở thành Expert trong Microservices! 🎉**

> "The best way to learn is to build something that you're passionate about."
