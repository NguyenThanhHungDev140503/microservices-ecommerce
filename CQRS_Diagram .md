# Tài Liệu Kiến Trúc CQRS - ProgCoder Shop Microservices

## 1. Tổng Quan Kiến Trúc

### Sơ Đồ Tổng Thể Hệ Thống

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY (YARP)                                │
│                    http://localhost:15009                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────┬───────────────┼───────────────┬─────────────┐
        ▼             ▼               ▼               ▼             ▼
   ┌─────────┐   ┌─────────┐    ┌─────────┐    ┌─────────┐   ┌─────────┐
   │ Catalog │   │  Order  │    │ Basket  │    │Discount │   │Inventory│
   │ Service │   │ Service │    │ Service │    │ Service │   │ Service │
   └────┬────┘   └────┬────┘    └────┬────┘    └────┬────┘   └────┬────┘
        │             │              │              │             │
        └─────────────┴──────────────┴──────────────┴─────────────┘
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                   ┌──────────┐              ┌──────────┐
                   │  Report  │              │  Search  │
                   │  Service │              │  Service │
                   └──────────┘              └──────────┘
```

### Bảng Tóm Tắt Các Service

| Service | Nhiệm Vụ Chính | Database | Pattern Đặc Biệt |
|---------|---------------|----------|------------------|
| **Catalog** | Quản lý sản phẩm, danh mục, thương hiệu | PostgreSQL | Outbox + Event Sourcing |
| **Order** | Xử lý đơn hàng | PostgreSQL | Outbox + Inbox + Saga |
| **Basket** | Giỏ hàng tạm thờI | MongoDB | Outbox Pattern |
| **Discount** | Mã giảm giá, coupon | MongoDB | CQRS Basic |
| **Inventory** | Quản lý tồn kho, đặt chỗ | PostgreSQL | Outbox + Inbox + Reservation |
| **Notification** | Gửi thông báo | MongoDB | Event Consumer |
| **Report** | Báo cáo, thống kê | MongoDB | Read Model Optimization |
| **Search** | Tìm kiếm sản phẩm | Elasticsearch | Event-driven Indexing |
| **Communication** | Real-time notifications | SignalR | Hub-based |

---

## 2. Cấu Trúc Clean Architecture Trong Mỗi Service

### Sơ Đồ Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         API LAYER                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  REST API   │  │  gRPC Svc   │  │     Carter Module       │  │
│  │  (Minimal)  │  │             │  │     (Endpoints)         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Commands   │  │   Queries   │  │    Event Handlers       │  │
│  │  (Writes)   │  │   (Reads)   │  │    (Domain/Integration) │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │    DTOs     │  │  Validators │  │       Services          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Entities   │  │   Events    │  │    Value Objects        │  │
│  │  (Aggregates)│ │ (Domain/Integration)│   (Money, Address)  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Repositories│  │  Exceptions │  │                         │  │
│  │ (Interfaces)│  │             │  │                         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   INFRASTRUCTURE LAYER                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  EF Core    │  │   MongoDB   │  │     MassTransit         │  │
│  │ (DbContext) │  │ (Repository)│  │     (Message Broker)    │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  gRPC Client│  │  API Client │  │    External Services    │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Bảng Mapping Các Thành Phần CQRS

| Layer | Thành Phần Write Side | Thành Phần Read Side | Mục Đích |
|-------|----------------------|---------------------|----------|
| **API** | Command Endpoints | Query Endpoints | Nhận request từ client |
| **Application** | Command + CommandHandler | Query + QueryHandler | Xử lý business logic |
| **Domain** | Aggregate + Domain Events | - | Đảm bảo business rules |
| **Infrastructure** | Repository Implementation | Projection/Read Model | Persist dữ liệu |

---

## 3. Cơ Chế Đồng Bộ Dữ Liệu Giữa Write và Read Models

### Sơ Đồ Luồng Dữ Liệu Tổng Thể (Outbox + Inbox Pattern)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              WRITE MODEL                                         │
│                                                                                  │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐      │
│   │   Command    │─────▶│   Aggregate  │─────▶│   Domain Event Raised    │      │
│   │   Handler    │      │   (Entity)   │      │   (OrderCreatedEvent)    │      │
│   └──────────────┘      └──────────────┘      └────────────┬─────────────┘      │
│                                                             │                    │
│                                                             ▼                    │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐      │
│   │  SaveChanges │◀─────│   DbContext  │◀─────│  DispatchDomainEvents    │      │
│   │   (EF Core)  │      │              │      │      Interceptor         │      │
│   └──────────────┘      └──────────────┘      └────────────┬─────────────┘      │
│                                                             │                    │
│                                                             ▼                    │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐      │
│   │   Outbox     │◀─────│  Transaction │      │  Save Event to Outbox    │      │
│   │   Messages   │      │   Commit     │      │      (Same Transaction)   │      │
│   │  (Pending)   │      │              │      │                          │      │
│   └──────┬───────┘      └──────────────┘      └──────────────────────────┘      │
│          │                                                                       │
└──────────┼───────────────────────────────────────────────────────────────────────┘
           │
           │ Outbox Processor (Background Service)
           │ Polling + Publish to Message Broker
           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MESSAGE BROKER                                         │
│                         (MassTransit + RabbitMQ)                                 │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐    │
│   │                    Exchange / Topic                                      │    │
│   │   (e.g., order-created, stock-changed, basket-checkout)                 │    │
│   └───────────────────────────────────┬─────────────────────────────────────┘    │
│                                       │                                          │
└───────────────────────────────────────┼──────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              READ MODEL                                          │
│                                                                                  │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐      │
│   │   Inbox      │◀─────│   Consumer   │◀─────│   Subscribe to Topic     │      │
│   │   Check      │      │   (Worker)   │      │   (MassTransit Consumer) │      │
│   │(Deduplication)│     │              │      │                          │      │
│   └──────┬───────┘      └──────────────┘      └──────────────────────────┘      │
│          │                                                                       │
│          │ Nếu chưa xử lý                                                       │
│          ▼                                                                       │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐      │
│   │  Integration │─────▶│  Event       │─────▶│   Update Read Model      │      │
│   │  Event Handler      │  Handler     │      │   (Projection)           │      │
│   └──────────────┘      └──────────────┘      └────────────┬─────────────┘      │
│                                                             │                    │
│                                                             ▼                    │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐      │
│   │   Mark as    │◀─────│   Inbox      │◀─────│   Save to Database       │      │
│   │   Processed  │      │   Message    │      │   (Mongo/Elasticsearch)  │      │
│   └──────────────┘      └──────────────┘      └──────────────────────────┘      │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Flowchart Xử Lý Outbox Pattern

```
┌─────────────────────────────────────────────────────────────┐
│  START: OutboxBackgroundService                             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  1. POLL: Query Outbox Messages với trạng thái PENDING      │
│     (Batch size: configured, mặc định 100)                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Có message nào không?                                      │
└──────────┬────────────────────────────┬─────────────────────┘
           │ YES                        │ NO
           ▼                            ▼
┌──────────────────────┐      ┌───────────────────────────────┐
│ 2. FOR EACH message  │      │ 5. SLEEP: Delay theo config   │
│    trong batch       │      │    (mặc định 10 giây)         │
└──────────┬───────────┘      └───────────┬───────────────────┘
           │                              │
           ▼                              │
┌────────────────────────────────────┐    │
│ 3. PUBLISH: Gửi event lên RabbitMQ │    │
│    qua MassTransit                 │    │
└──────────┬─────────────────────────┘    │
           │                              │
           ▼                              │
┌────────────────────────────────────┐    │
│ 4. UPDATE: Đánh dấu message là      │    │
│    PROCESSED hoặc FAILED           │    │
└──────────┬─────────────────────────┘    │
           │                              │
           └──────────────┬───────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  6. REPEAT: Quay lại bước 1 (POLL)                          │
└─────────────────────────────────────────────────────────────┘
```

### Flowchart Xử Lý Inbox Pattern

```
┌─────────────────────────────────────────────────────────────┐
│  START: Integration Event Consumer                          │
│  (e.g., OrderCreatedIntegrationEventHandler)                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  1. RECEIVE: Nhận event từ Message Broker                   │
│     (MassTransit Consumer)                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  2. CHECK: Kiểm tra Inbox Messages                          │
│     SELECT * WHERE MessageId = @MessageId                   │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│  Message đã tồn tại chưa?                                   │
└──────────┬────────────────────────────┬─────────────────────┘
           │ YES                        │ NO
           ▼                            ▼
┌──────────────────────┐      ┌───────────────────────────────┐
│  3a. SKIP: Bỏ qua    │      │ 3b. INSERT: Lưu message vào   │
│     (Idempotency)    │      │     Inbox với trạng thái      │
│                      │      │     PENDING                   │
└──────────────────────┘      └───────────┬───────────────────┘
                                          │
                                          ▼
                              ┌───────────────────────────────┐
                              │ 4. PROCESS: Xử lý business    │
                              │    logic của event            │
                              │    (Update read model)        │
                              └───────────┬───────────────────┘
                                          │
                                          ▼
                              ┌───────────────────────────────┐
                              │ 5. UPDATE: Đánh dấu Inbox     │
                              │    message là COMPLETED       │
                              └───────────────────────────────┘
```

### Bảng Tóm Tắt Các Integration Events

| Event Name | Publisher | Subscribers | Mục Đích |
|------------|-----------|-------------|----------|
| `BasketCheckoutIntegrationEvent` | Basket | Order | Chuyển giỏ hàng thành đơn hàng |
| `OrderCreatedIntegrationEvent` | Order | Inventory, Communication | Trừ tồn kho, gửi thông báo |
| `OrderCancelledIntegrationEvent` | Order | Inventory | Hoàn trả tồn kho |
| `OrderDeliveredIntegrationEvent` | Order | Inventory | Xác nhận giao hàng |
| `StockChangedIntegrationEvent` | Inventory | Catalog, Search | Cập nhật số lượng sản phẩm |
| `ReservationExpiredIntegrationEvent` | Inventory | Order | Hết hạn đặt chỗ |
| `UpsertedProductIntegrationEvent` | Catalog | Search, Notification | Cập nhật search index |
| `DeletedUnPublishedProductIntegrationEvent` | Catalog | Search | Xóa khỏi search index |

---

## 4. Sơ Đồ Luồng Xử Lý Đơn Hàng (Order Flow)

### Sequence Diagram: Từ Basket đến Order

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Client  │     │ Basket  │     │  Order  │     │Inventory│     │Payment* │
└────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
     │               │               │               │               │
     │  1. Checkout  │               │               │               │
     │──────────────▶│               │               │               │
     │               │               │               │               │
     │               │ 2. Publish    │               │               │
     │               │    BasketCheckoutEvent        │               │
     │               │──────────────▶│               │               │
     │               │               │               │               │
     │               │               │ 3. Reserve    │               │
     │               │               │    Inventory  │               │
     │               │               │──────────────▶│               │
     │               │               │               │               │
     │               │               │ 4. Reserved   │               │
     │               │               │◀──────────────│               │
     │               │               │               │               │
     │               │               │ 5. Create     │               │
     │               │               │    Order      │               │
     │               │               │────┐          │               │
     │               │               │    │          │               │
     │               │               │◀───┘          │               │
     │               │               │               │               │
     │               │               │ 6. Publish    │               │
     │               │               │    OrderCreatedEvent          │
     │               │               │──────────────▶│               │
     │               │               │               │               │
     │               │               │               │ 7. Confirm    │
     │               │               │               │    Stock      │
     │               │               │               │               │
     │  8. Response  │               │               │               │
     │◀──────────────┼───────────────┼───────────────┼───────────────┘
     │               │               │               │
```

### Bảng Trạng Thái Đơn Hàng

| Trạng Thái | Mô Tả | Event Kích Hoạt |
|------------|-------|-----------------|
| **Pending** | Đơn hàng vừa tạo | `OrderCreatedDomainEvent` |
| **Reserved** | Đã đặt chỗ tồn kho | `StockReservedIntegrationEvent` |
| **Paid** | Đã thanh toán | `PaymentCompletedIntegrationEvent` |
| **Shipped** | Đang giao hàng | `OrderShippedDomainEvent` |
| **Delivered** | Đã giao hàng | `OrderDeliveredDomainEvent` |
| **Cancelled** | Đã hủy | `OrderCancelledDomainEvent` |

---

## 5. Sơ Đồ Worker Services

### Kiến Trúc Worker Outbox

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WORKER OUTBOX SERVICE                                │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │              OutboxBackgroundService (Hosted Service)               │    │
│   │                                                                     │    │
│   │   ┌─────────────┐      ┌─────────────┐      ┌─────────────────┐    │    │
│   │   │   Timer     │─────▶│   Execute   │─────▶│  OutboxProcessor│    │    │
│   │   │  (Interval) │      │   Async     │      │                 │    │    │
│   │   └─────────────┘      └─────────────┘      └────────┬────────┘    │    │
│   │                                                      │             │    │
│   └──────────────────────────────────────────────────────┼─────────────┘    │
│                                                          │                  │
│                                                          ▼                  │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │                      OutboxProcessor                                │    │
│   │                                                                     │    │
│   │   ┌─────────────┐      ┌─────────────┐      ┌─────────────────┐    │    │
│   │   │  Get Pending│─────▶│   Publish   │─────▶│  Mark Processed │    │    │
│   │   │  Messages   │      │   Events    │      │  (MassTransit)  │    │    │
│   │   └─────────────┘      └─────────────┘      └─────────────────┘    │    │
│   │                                                                     │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bảng Tóm Tắt Các Worker Services

| Worker Service | Nhiệm Vụ | Database | Trigger |
|---------------|----------|----------|---------|
| **Basket.Worker.Outbox** | Publish basket events | MongoDB | Background polling |
| **Catalog.Worker.Outbox** | Publish product events | PostgreSQL | Background polling |
| **Catalog.Worker.Consumer** | Xử lý stock changed events | - | Event-driven |
| **Inventory.Worker.Outbox** | Publish inventory events | PostgreSQL | Background polling |
| **Inventory.Worker.Consumer** | Xử lý order events | - | Event-driven |
| **Order.Worker.Outbox** | Publish order events | PostgreSQL | Background polling |
| **Order.Worker.Consumer** | Xử lý basket checkout, reservation expired | - | Event-driven |
| **Notification.Worker.Consumer** | Xử lý product events để gửi thông báo | - | Event-driven |
| **Search.Worker.Consumer** | Cập nhật Elasticsearch index | - | Event-driven |

---

## 6. Sơ Đồ Communication Patterns

### Inter-service Communication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SYNCHRONOUS (gRPC)                                   │
│                         (Request/Response)                                   │
│                                                                              │
│    Service A              gRPC Channel               Service B               │
│   ┌─────────┐          ┌─────────────┐          ┌─────────┐                  │
│   │  Client │─────────▶│   HTTP/2    │─────────▶│ Server  │                  │
│   │         │◀─────────│   (Proto)   │◀─────────│         │                  │
│   └─────────┘          └─────────────┘          └─────────┘                  │
│                                                                              │
│   Use cases:                                                                 │
│   - Basket ──gRPC──▶ Catalog (Get Product Info)                             │
│   - Order ───gRPC──▶ Discount (Evaluate Coupon)                             │
│   - Order ───gRPC──▶ Catalog (Get Product Details)                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        ASYNCHRONOUS (Message Broker)                         │
│                         (Pub/Sub Pattern)                                    │
│                                                                              │
│                         ┌─────────────────┐                                  │
│                         │   RabbitMQ      │                                  │
│    Service A            │   (Exchange)    │            Service B             │
│   ┌─────────┐           │                 │           ┌─────────┐            │
│   │ Publish │──────────▶│    order.*      │──────────▶│ Consume │            │
│   │  Event  │           │    catalog.*    │           │  Event  │            │
│   └─────────┘           │    inventory.*  │           └─────────┘            │
│                         │                 │                                  │
│                         └─────────────────┘                                  │
│                                                                              │
│   Use cases:                                                                 │
│   - Basket ──Event──▶ Order (Checkout)                                      │
│   - Order ───Event──▶ Inventory (Reserve/Cancel/Commit)                     │
│   - Order ───Event──▶ Communication (Notification)                          │
│   - Catalog ─Event──▶ Search (Index Update)                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bảng So Sánh Synchronous vs Asynchronous

| Tiêu Chí | Synchronous (gRPC) | Asynchronous (Events) |
|----------|-------------------|----------------------|
| **Latency** | Thấp (HTTP/2) | Cao (queue processing) |
| **Consistency** | Strong | Eventual |
| **Coupling** | Tight (phụ thuộc service online) | Loose (fire and forget) |
| **Use Case** | Query data, Validate | State changes, Notifications |
| **Example** | GetProductPrice | OrderCreated |

---

## 7. Tóm Tắt Các Design Patterns Được Sử Dụng

| Pattern | Mục Đích | Nơi Áp Dụng |
|---------|----------|-------------|
| **CQRS** | Tách biệt Read/Write models | Tất cả services |
| **Outbox** | Đảm bảo delivery của events | PostgreSQL services |
| **Inbox** | Deduplicate consumed events | PostgreSQL services |
| **Saga** | Quản lý distributed transaction | Order + Inventory |
| **Event Sourcing** | Lưu trữ state như sequence of events | Catalog, Order |
| **Repository** | Abstract data access | Domain layer |
| **Unit of Work** | Quản lý transaction | Infrastructure layer |
| **Mediator** | Decouple command/query handlers | Application layer |

---

## 8. Cấu Trúc Project Theo Clean Architecture

```
src/
├── Services/
│   └── [ServiceName]/
│       ├── Api/
│       │   └── [Service].Api/
│       │       ├── Endpoints/          # Carter modules (Minimal API)
│       │       └── Program.cs          # DI registration
│       │
│       ├── Core/
│       │   ├── [Service].Application/  # Business logic, CQRS handlers
│       │   │   ├── Features/           # Commands & Queries
│       │   │   ├── EventHandlers/      # Domain & Integration handlers
│       │   │   └── Repositories/       # Interfaces
│       │   │
│       │   └── [Service].Domain/       # Entities, Events, Value Objects
│       │       ├── Entities/
│       │       ├── Events/
│       │       └── ValueObjects/
│       │
│       ├── Infrastructure/
│       │   └── [Service].Infrastructure/
│       │       ├── Data/               # DbContext, Migrations
│       │       ├── Repositories/       # Implementations
│       │       └── Services/           # External service clients
│       │
│       └── Worker/
│           ├── [Service].Worker.Outbox/     # Background publisher
│           └── [Service].Worker.Consumer/   # Event consumers
│
└── Shared/
    ├── BuildingBlocks/           # Common CQRS infrastructure
    ├── Common/                   # Constants, Extensions
    └── Contracts/                # gRPC Protos
```

---

*Document được tạo từ phân tích codebase ProgCoder Shop Microservices*
