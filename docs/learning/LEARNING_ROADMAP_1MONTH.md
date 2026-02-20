# 🚀 Lộ Trình Học Tập .NET Microservices - 1 Tháng
## Dự án: Progcoder Shop Microservices

> **Mục tiêu:** Từ người mới hiểu .NET và REST API → Nắm vững kiến trúc Microservices thực tế
>
> **Thời gian:** 4 tuần (1 tháng)
>
> **Chiến lược:** Tập trung vào 80/20 rule - 20% kiến thức quan trọng nhất mang lại 80% giá trị
>
> **Service chính để học:** Order Service (đầy đủ nhất, phức tạp nhất)

---

## 📋 Tổng Quan Kiến Trúc (Overview)

```
┌─────────────────────────────────────────────────────┐
│                  YARP Gateway                       │
│          (API Gateway - "Người gác cổng")            │
└─────────────┬───────────────────────────────────────┘
              │
    ┌─────────┴─────────┬───────────────┬────────────┐
    │                   │               │            │
    ▼                   ▼               ▼            ▼
┌────────┐         ┌──────────┐   ┌──────────┐  ┌─────────┐
│ Order  │◄──────►│ Inventory│   │ Catalog  │  │ Basket  │
│ Service│  gRPC   │ Service  │   │ Service  │  │ Service │
└───┬────┘         └──────────┘   └──────────┘  └─────────┘
    │                    │               │
    └───────┬────────────┘               │
            │                            │
            ▼                            ▼
      ┌──────────┐                  ┌─────────┐
      │ RabbitMQ │                  │  Redis   │
      │+ MassTransit               │  Cache   │
      └──────────┘                  └─────────┘
```

### 4 Weeks Roadmap

| Week | Chủ đề | Output |
|------|--------|--------|
| Week 1 | Clean Architecture & DDD | Hiểu cấu trúc 1 service chuẩn |
| Week 2 | Giao tiếp Sync & Async | gRPC + RabbitMQ + MassTransit |
| Week 3 | Outbox Pattern & Redis | Độ tin cậy dữ liệu + Caching |
| Week 4 | API Gateway & Docker | Triển khai full stack |

---

## Week 1: Clean Architecture & DDD (3.5 ngày)
### 🎯 Mục tiêu
- Hiểu cấu trúc thư mục chuẩn của một Microservice
- Nắm vững DDD: Entities, Value Objects, Aggregates
- Thực hành MediatR CQRS

### 📚 Day 1: Cấu trúc Clean Architecture

**🔍 Khám phá Order Service**
- **Mở thư mục:** `src/Services/Order`
- **Vẽ sơ đồ:**
  ```
  Order/
  ├── Core/
  │   ├── Order.Domain/         ← Entities, Value Objects (NO dependencies)
  │   ├── Order.Application/    ← Use Cases (MediatR CQRS)
  │   └── Order.Infrastructure/ ← EF Core, Repositories
  ├── Api/
  │   ├── Order.Api/            ← REST Controllers
  │   └──.Order.Grpc/           ← gRPC Services
  └── Worker/
      ├── Order.Worker.Consumer/     ← RabbitMQ Consumers
      └── Order.Worker.Outbox/       ← Outbox Pattern
  ```

**✅ Task 1.1: [ ] Hiểu Dependency Direction**
- **File xem:** `src/Services/Order/Core/Order.Domain/Entities/Order.cs`
- **Câu hỏi:** Tại sao Domain layer không được reference Infrastructure?
- **Output:** Viết note giải thích Dependency Inversion Principle

**✅ Task 1.2: [ ] Tìm hiểu Entities**
- **File xem:** `Order.cs`, `OrderItem.cs`
- **Câu hỏi:**
  - đâu là Aggregate Root? (Order hay OrderItem?)
  - Tại sao không thể truy cập OrderItem trực tiếp?
- **Output:** Vẽ diagram Aggregates trong Order Service

---

### 📚 Day 2: Application Layer & CQRS

**🔍 Khám phá MediatR**
- **File xem:** `src/Services/Order/Core/Order.Application/Commands/`
  - `CreateOrderCommand.cs` - Request
  - `CreateOrderCommandHandler.cs` - Handler

**✅ Task 2.1: [ ] Hiểu CQRS**
- **Câu hỏi:**
  - Tại sao tách Command và Query?
  - MediatR dùng Reflection để làm gì?
- **Output:** Vẽ flow:
  ```
  Controller → MediatR.Send() → Handler → Repository → DB
  ```

**✅ Task 2.2: [ ] Tìm hiểu Validation**
- **File xem:** `src/Services/Order/Core/Order.Application/Validators/`
  - `CreateOrderCommandValidator.cs`
- **Câu hỏi:**
  - Tại sao dùng FluentValidation thay vì `[Required]`?
- **Output:** Thêm một validation rule mới cho CreateOrderCommand

---

### 📚 Day 3: Infrastructure & API Layer

**🔍 Khám phá EF Core**
- **File xem:** `src/Services/Order/Core/Order.Infrastructure/Data/OrderDbContext.cs`
- **File xem:** `src/Services/Order/Core/Order.Infrastructure/Configurations/`

**✅ Task 3.1: [ ] Hiểu Repository Pattern**
- **File xem:** `IOrderRepository.cs`, `OrderRepository.cs`
- **Câu hỏi:**
  - Tại sao cần Repository thay vì dùng DbContext trực tiếp?
  - Repository có chứa business logic không?

**✅ Task 3.2: [ ] Hiểu API Layer**
- **File xem:** `src/Services/Order/Api/Order.Api/Program.cs`
- **Câu hỏi:**
  - Tại sao API layer nên "thin"?
  - Dependency Injection đăng ký như thế nào?

---

### 📚 Day 3.5: Assignment Week 1

**🎯 Project: Tạo mini Order Service**
- [ ] Tạo một simple service với Clean Architecture:
  - Domain: Product entity
  - Application: CreateProductCommand, GetProductsQuery
  - Infrastructure: EF Core with In-Memory DB
  - API: Minimal APIs
- [ ] Test bằng Swagger UI
- [ ] Ghi note những phần chưa hiểu

---

## Week 2: Giao Tiếp Giữa Services (Sync & Async) (4 ngày)
### 🎯 Mục tiêu
- Thực hành gRPC (giao tiếp đồng bộ nhanh)
- Hiểu RabbitMQ + MassTransit (giao tiếp bất đồng bộ)
- Biết khi nào dùng Sync, khi nào dùng Async

### 📚 Day 4: gRPC Fundamentals

**🔍 Khám phá gRPC**
- **File xem:** `src/Services/Order/Api/Order.Grpc/`
  - `Protos/order.proto`
  - `OrderGrpcService.cs`

**✅ Task 4.1: [ ] Hiểu Protocol Buffers**
- **Bài tập:**
  - Đọc file `.proto`, hiểu message syntax
  - So sánh Protobuf vs JSON (size, speed, type-safe)
- **Output:** Vẽ diagram gRPC call flow

**✅ Task 4.2: [ ] Test gRPC**
- **Tool:** Dùng Postman (hỗ trợ gRPC) hoặc `grpcurl`
- **Bài tập:** Gọi `GetOrderById` gRPC endpoint
- **Output:** Screenshot kết quả

---

### 📚 Day 5: RabbitMQ & MassTransit - Publish Events

**🔍 Khám phá Message Broker**
- **File xem:** `src/Shared/EventSourcing/MassTransit/Extensions.cs`
- **Run:** `docker-compose up -d rabbitmq`
- **UI:** `http://localhost:15672` (username: guest, password: guest)

**✅ Task 5.1: [ ] Hiểu RabbitMQ UI**
- **Bài tập:**
  - Xem Queues, Exchanges, Bindings
  - Gửi test message thủ công
- **Output:** Ghi note về các khái niệm: Queue, Exchange, Binding, Routing Key

**✅ Task 5.2: [ ] Publish Events với MassTransit**
- **File xem:** `src/Shared/Contracts/Order.Contract/`
  - `OrderCreatedEvent.cs`
- **Bài tập:** Tìm code dùng `IPublishEndpoint`
- **Câu hỏi:**
  - Tại sao Contracts nằm trong Shared project?
- **Output:** Vẽ diagram publish event:
  ```
  Service A → Publish Event → RabbitMQ Exchange → Queue → Service B
  ```

---

### 📚 Day 6: MassTransit - Consume Events

**🔍 Khám phá Consumers**
- **File xem:** `src/Services/Order/Worker/Order.Worker.Consumer/`
  - Các class implement `IConsumer<T>`

**✅ Task 6.1: [ ] Hiểu Consumer Lifecycle**
- **Bài tập:** Vẽ flow:
  ```
  RabbitMQ delivers message → MassTransit deserializes → Consumer.Consume() → Ack/Nack
  ```
- **Câu hỏi:**
  - MassTransit có retry lại khi Consumer throw exception không?

**✅ Task 6.2: [ ] Retry Policies**
- **File xem:** `src/Shared/EventSourcing/MassTransit/Extensions.cs`
- **Bài tập:** Tìm hiểu:
  - Immediate retry
  - Exponential backoff (thời gian tăng dần)
  - Số lần retry mặc định?
- **Output:** Config retry policy cho một Consumer

---

### 📚 Day 7: Assignment Week 2

**🎯 Project: Event-Driven Communication**
- [ ] Tạo một Integration Event mới: `OrderCancelledEvent`
- [ ] Implement Consumer trong Inventory Service (tăng lại inventory khi order bị hủy)
- [ ] Test:
  ```bash
  # Step 1: Create order
  POST /api/orders

  # Step 2: Watch RabbitMQ UI - message published

  # Step 3: Check Inventory Service logs - message consumed
  ```
- [ ] Đọc: `docs/RabbitMQ/RabbitMQ_MassTransit_Onboarding_Guide_v2.md` (section 1-3)
- [ ] Ghi note về Saga Pattern (optional - đọc hiểu thôi)

---

## Week 3: Độ Tin Cậy & Hiệu Năng (3.5 ngày)
### 🎯 Mục tiêu
- Master Outbox Pattern (quan trọng nhất!)
- Thực hành Redis Caching
- Hiểu Polly Resilience

### 📚 Day 8: Outbox Pattern - Giải quyết vấn đề Distributed Transactions

**🔍 Hiểu vấn đề**
- **Câu hỏi:**
  - Lưu Order vào DB thành công → Publish Event thất bại → Kết quả?
  - Publish Event thành công → Lưu DB thất bại → Kết quả?
  - System sẽ ở trạng thái inconsistent (không nhất quán)

**✅ Task 8.1: [ ] Nghiên cứu Outbox Pattern**
- **File xem:** `src/Services/Order/Worker/Order.Worker.Outbox/`
- **Bài tập:** Vẽ flow:
  ```
  1. Save Order + OutboxMessage (same DB transaction)
  2. OutboxProcessor reads pending messages
  3. Publish Event to RabbitMQ
  4. Mark OutboxMessage as Processed
  ```

**✅ Task 8.2: [ ] Hiểu Idempotency**
- **Câu hỏi:**
  - OutboxProcessor có thể process cùng message 2 lần không?
  - Làm sao đảm bảo event chỉ được publish 1 lần?
  - Làm sao đảm bảo Consumer process event 2 lần vẫn ra kết quả như nhau?
- **Output:** Viết note về Idempotency trong Outbox & Consumer

---

### 📚 Day 9: Redis Caching

**🔍 Khám phá Basket Service (dùng Redis)**
- **File xem:** `src/Services/Basket/Core/Basket.Infrastructure/`
- **Run:** `docker-compose up -d redis`
- **Tool:** Redis CLI: `redis-cli`

**✅ Task 9.1: [ ] Hiểu Redis Data Structures**
- **Bài tập:**
  - Connect Redis CLI: `redis-cli`
  - Test commands:
    ```bash
    SET user:1:basket '{"items":[]}' EX 3600  # Expire sau 1 giờ
    GET user:1:basket
    DEL user:1:basket
    KEYS user:*
    ```
- **Output:** Note về 5 data types: String, Hash, List, Set, Sorted Set

**✅ Task 9.2: [ ] Cache-Aside Pattern**
- **Bài tập:** Vẽ flow:
  ```
  Request → Check Redis
            → Hit? Return
            → Miss? Query DB → Save to Redis → Return
  ```
- **File xem:** Tìm code dùng Redis trong Basket Service

---

### 📚 Day 10: Polly Resilience

**🔍 Hiểu Fault Tolerance**
- **Câu hỏi:**
  - Database bị slow → System làm gì?
  - External API bị down → System làm gì?

**✅ Task 10.1: [ ] Cấu hình Polly**
- **File xem:** Tìm `AddPolicyHandler` trong `Program.cs` hoặc `Extensions.cs`
- **Bài tập:** Tìm hiểu:
  - Retry Policy: Thử lại N lần
  - Circuit Breaker: Ngắt cầu dao sau N lần fail
  - Timeout Policy: Đợi quá lâu → timeout

**✅ Task 10.2: [ ] Test Circuit Breaker**
- **Bài tập:**
  1. Configure Circuit Breaker: 3 failures → Open for 30 seconds
  2. Shutdown một service (ví dụ: Inventory)
  3. Gọi API liên tục → Xem logs thấy Circuit Breaker Open
  4. Start service lại → Circuit Breaker Half-Open → Closed
- **Output:** Screenshot logs hiển thị Circuit Breaker state

---

### 📚 Day 10.5: Assignment Week 3

**🎯 Project: Resilient System**
- [ ] Implement Outbox Pattern cho một simple service (dùng SQLite + MassTransit)
- [ ] Add Redis caching cho một endpoint (ví dụ: GetProducts)
- [ ] Configure Polly retry cho HTTP client
- [ ] Test by:
  - Stop Database → System vẫn hoạt động nhờ retry
  - Stop RabbitMQ → Outbox saves message, publish sau
  - Stop Redis → Fallback to Database
- [ ] Đọc: `docs/RabbitMQ/RabbitMQ_MassTransit_Architecture.md` (phần Outbox)

---

## Week 4: API Gateway & Deployment (3.5 ngày)
### 🎯 Mục tiêu
- Hiểu API Gateway pattern
- Docker & Containerization
- Deploy toàn bộ stack

### 📚 Day 11: YARP API Gateway

**🔍 Khám phá Gateway**
- **File xem:** `src/ApiGateway/YarpApiGateway/`
  - `appsettings.json`
  - `Program.cs`

**✅ Task 11.1: [ ] Hiểu Routing**
- **Bài tập:** Đọc config trong `appsettings.json`:
  ```json
  "Routes": {
    "OrderRoute": {
      "ClusterId": "OrderCluster",
      "Match": { "Path": "/api/orders/{**catch-all}" }
    }
  },
  "Clusters": {
    "OrderCluster": {
      "Destinations": {
        "OrderApi": { "Address": "http://order-api:5000" }
      }
    }
  }
  ```
- **Câu hỏi:**
  - Tại sao cần Gateway? (Hide internal services, Rate limiting, Auth, etc.)
  - Client gọi `http://gateway/api/orders` thay vì `http://order-api:5000/api/orders`?

**✅ Task 11.2: [ ] Test Gateway Routing**
- **Bài tập:**
  ```bash
  # Call through Gateway
  curl http://localhost:5000/api/orders

  # Compare logs - Gateway forwards request to Order Service
  docker-compose logs -f yarp-gateway
  docker-compose logs -f order-api
  ```
- **Output:** Screenshot logs hiển thị routing

---

### 📚 Day 12: Docker Fundamentals

**🔍 Hiểu Containerization**
- **File xem:** `Dockerfile` trong các services

**✅ Task 12.1: [ ] Đọc Dockerfile**
- **Bài tập:** Hiểu từng dòng:
  ```dockerfile
  FROM mcr.microsoft.com/dotnet/aspnet:8.0
  WORKDIR /app
  COPY bin/Release/net8.0/publish .
  ENTRYPOINT ["dotnet", "Order.Api.dll"]
  ```
- **Câu hỏi:**
  - Container khác với VM thế nào?
  - Tại sao cần multi-stage build?

**✅ Task 12.2: [ ] Docker Compose**
- **File xem:** `docker-compose.yml` (root folder)
- **Bài tập:**
  - Hiểu `services`, `networks`, `volumes`
  - Hiểu `depends_on` và `healthcheck`
- **Câu hỏi:**
  - Làm sao services connect với nhau? (Container network)
  - Environment variables dùng để làm gì?

---

### 📚 Day 13: Deploy Full Stack

**🔍 Triển khai toàn bộ hệ thống**
- **Run:** `docker-compose up -d`
- **Check:**
  ```bash
  # Check all services
  docker-compose ps

  # Check logs
  docker-compose logs -f

  # Test each service
  curl http://localhost:5000/api/orders  # Gateway
  curl http://localhost:5001/api/orders  # Order API
  curl http://localhost:15672            # RabbitMQ UI
  ```

**✅ Task 13.1: [ ] Test End-to-End Flow**
- **Bài tập:** Tạo order và theo dõi flow:
  ```
  1. POST http://gateway/api/orders
  2. Order Service saves to DB + Outbox
  3. OutboxProcessor publishes OrderCreatedEvent
  4. Inventory Service receives event → reserves inventory
  5. Notification Service receives event → sends email
  ```
- **Tool:**
  - RabbitMQ UI: http://localhost:15672
  - Logs: `docker-compose logs -f order-worker-consumer`

**✅ Task 13.2: [ ] Troubleshooting**
- **Bài tập:** Tạo lỗi và fix:
  - Stop Database → System có retry?
  - Stop RabbitMQ → Outbox saves message?
  - Stop Redis → Cache fallback?
- **Output:** Note về troubleshooting steps

---

### 📚 Day 14: Assignment Week 4 - Final Project

**🎯 Capstone Project: Deploy Microservices Coffee Shop**
- [ ] Create simple services:
  - Menu Service (CRUD drinks)
  - Order Service (CreateOrder, gRPC)
  - Notification Service (Send email, RabbitMQ)
- [ ] Implement patterns:
  - Clean Architecture (DDD)
  - CQRS with MediatR
  - gRPC between services
  - RabbitMQ + MassTransit for events
  - Outbox Pattern for reliability
  - Redis for caching
  - Polly for resilience
- [ ] Deploy:
  - Dockerfile cho mỗi service
  - docker-compose.yml orchestrate all
  - YARP Gateway routing
- [ ] Test end-to-end:
  - Client creates order via Gateway
  - Order Service saves DB + Outbox
  - Menu Service checks inventory (gRPC)
  - Notification Service sends confirmation (RabbitMQ)
- [ ] Documentation:
  - README.md với architecture diagram
  - API documentation (Swagger)
  - Docker commands to run

---

## 📊 Milestones Checkpoints

### ✅ Milestone 1 (End of Week 1)
- [ ] Có thể giải thích cấu trúc Order Service
- [ ] Có thể tạo service mới với Clean Architecture
- [ ] Thực hành MediatR CQRS

### ✅ Milestone 2 (End of Week 2)
- [ ] Có thể create gRPC service & client
- [ ] Có thể publish/consume events với MassTransit
- [ ] Hiểu khi nào dùng sync vs async

### ✅ Milestone 3 (End of Week 3)
- [ ] Có thể implement Outbox Pattern
- [ ] Có thể dùng Redis caching
- [ ] Có thể configure Polly resilience

### ✅ Milestone 4 (End of Week 4)
- [ ] Có thể deploy full stack với Docker Compose
- [ ] Có thể configure API Gateway
- [ ] Có thể debug production issues

---

## 🚀 Tips Học Tập Hiệu Quả (4 Weeks)

### 1. Learn by Doing - Thực hành nhiều hơn lý thuyết
- Đừng chỉ đọc code → Modify code
- Tạo branch mới cho mỗi task
- Commit với message mô tả task
- 60% coding, 20% reading, 20% debugging

### 2. Focus on High-Impact Topics - Tập trung vào quan trọng nhất
| Topic | Priority | Time |
|-------|----------|------|
| Clean Architecture, DDD | 🔥 High | Week 1 |
| gRPC + RabbitMQ | 🔥 High | Week 2 |
| Outbox Pattern | 🔥 High | Week 3 |
| Redis, Polly | ⚡ Medium | Week 3 |
| Docker, Gateway | ⚡ Medium | Week 4 |
| Monitoring, CI/CD | ❓ Low | Sau này |

### 3. Debug Systemically - Debug có hệ thống
Khi gặp lỗi:
1. Check logs: `docker-compose logs -f <service-name>`
2. Check RabbitMQ UI: xem queues có message không?
3. Check Database: data có inconsistency không?
4. Test từng component riêng lẻ

### 4. Draw Diagrams - Vẽ sơ đồ
- Architecture diagram (components & relationships)
- Sequence diagram (request/response flow)
- State diagram (circuit breaker, saga)
- Lưu diagram trong repo

### 5. Take Notes - Ghi chép
- Tạo `learning-notes.md` trong mỗi service
- Note những pattern đã học
- Note những bug đã gặp và cách fix
- Update diagram khi hiểu rõ hơn

### 6. Use Resources Sapients - Sử dụng tài liệu khôn ngoan
- **Project docs:**
  - `docs/RabbitMQ/RabbitMQ_MassTransit_Onboarding_Guide_v2.md`
  - `docs/RabbitMQ/RabbitMQ_MassTransit_Architecture.md`
- **External:**
  - MassTransit: https://masstransit.io/
  - gRPC: https://grpc.io/docs/languages/dotnet/
  - YARP: https://microsoft.github.io/reverse-proxy/

---

## 📅 Weekly Schedule Gợi Ý

### Week 1 Schedule
| Day | Morning (2h) | Afternoon (2h) | Evening (1h) |
|-----|---------------|-----------------|---------------|
| Day 1 | Study Architecture | Explore Order Domain | Draw diagram |
| Day 2 | Study CQRS, MediatR | Practice Validation | Note take |
| Day 3 | Study Infrastructure | Practice EF Core | Debug |
| Day 3.5 | Assignment | - | Review |

### Week 2 Schedule
| Day | Morning (2h) | Afternoon (2h) | Evening (1h) |
|-----|---------------|-----------------|---------------|
| Day 4 | Study gRPC | Test gRPC | Note |
| Day 5 | Study RabbitMQ | Publish Events | Play with UI |
| Day 6 | Study Consumers | Retry Policies | Debug |
| Day 7 | Assignment | - | Review |

### Week 3 Schedule
| Day | Morning (2h) | Afternoon (2h) | Evening (1h) |
|-----|---------------|-----------------|---------------|
| Day 8 | Study Outbox | Implement Outbox | Note |
| Day 9 | Study Redis | Practice Redis CLI | Cache pattern |
| Day 10 | Study Polly | Test Circuit Breaker | Debug |
| Day 10.5 | Assignment | - | Review |

### Week 4 Schedule
| Day | Morning (2h) | Afternoon (2h) | Evening (1h) |
|-----|---------------|-----------------|---------------|
| Day 11 | Study Gateway | Configure Routes | Test |
| Day 12 | Study Docker | Build images | Debug |
| Day 13 | Deploy Stack | Test E2E flow | Troubleshoot |
| Day 14 | Final Project | Deploy | Presentation |

**Total time per week: ~20-25 hours**
**Total time: ~80-100 hours trong 4 tuần**

---

## 🎯 Success Criteria - Bạn thành công khi...

### Technical Skills
- [ ] Có thể tạo microservice từ đầu với Clean Architecture
- [ ] Có thể implement gRPC communication
- [ ] Có thể implement event-driven architecture với RabbitMQ + MassTransit
- [ ] Có thể apply Outbox Pattern để đảm bảo consistency
- [ ] Có thể deploy stack với Docker Compose

### Soft Skills
- [ ] Có thể giải thích architecture cho người khác
- [ ] Có thể debug distributed system issues
- [ ] Có thể trade-off giữa các giải pháp (sync vs async, etc.)

### Portfolio
- [ ] GitHub repo với capstone project
- [ ] Architecture diagram
- [ ] Demo video hoặc screenshots
- [ ] Blogpost hoặc presentation slides

---

## 🆘 Need Help?

### Debug Common Issues
1. **RabbitMQ connection failed**
   - Check: `docker ps` (RabbitMQ running?)
   - Check: `docker-compose logs rabbitmq`
   - Fix: Restart RabbitMQ, check ports

2. **Database connection failed**
   - Check: `docker-compose ps` (SQL running?)
   - Check: Connection string in `appsettings.json`
   - Fix: Ensure services in same Docker network

3. **gRPC call failed**
   - Check: Protobuf file syntax
   - Check: Service registration in DI
   - Fix: Rebuild gRPC client

### Useful Commands
```bash
# Run all services
docker-compose up -d

# Check logs
docker-compose logs -f <service-name>

# Stop all services
docker-compose down

# Rebuild specific service
docker-compose up -d --build order-api

# Connect to container
docker exec -it <container-id> /bin/bash

# Test API
curl http://localhost:5000/api/orders
```

---

## 📚 Recommended Reading Order

### Week 1
1. Read `README.md` (project overview)
2. Read Clean Architecture notes (file này)
3. Read DDD concepts (online resources)

### Week 2
1. Read `docs/RabbitMQ/RabbitMQ_MassTransit_Onboarding_Guide_v2.md` (sections 1-4)
2. Read gRPC documentation (official site)

### Week 3
1. Read `docs/RabbitMQ/RabbitMQ_MassTransit_Architecture.md` (Outbox section)
2. Read Redis patterns (online resources)
3. Read Polly documentation (GitHub)

### Week 4
1. Read Docker documentation (official site)
2. Read YARP documentation (Microsoft)

---

## 🎉 Final Message

**Chúc bạn hoàn thành lộ trình 4 tuần!**

**Key takeaways:**
1. Microservices = Architecture + Patterns + Tools
2. Clean Architecture = Tách biệt Domain vs Infrastructure
3. Event-driven = Asynchronous communication with message broker
4. Outbox Pattern = Guaranteed message delivery
5. Docker = Easy deployment & scaling

**Next steps sau 4 tuần:**
- Deep dive into specific topics (Kafka, Kubernetes, etc.)
- Production monitoring (ELK Stack, Prometheus, Grafana)
- Advanced patterns (Event Sourcing, Saga Choreography)
- CI/CD pipeline (GitHub Actions, Azure DevOps)

**Remember:** Lộ trình này là starting point. Học mãi mãi!

> "The expert in anything was once a beginner."
> — Helen Hayes

---

**Good luck & happy coding! 🚀**
