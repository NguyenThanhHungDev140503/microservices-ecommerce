# Phan tich thiet ke Domain-Driven Design (DDD) - ProG Coder Shop Microservices

## 1. Tong quan kien truc

Du an **ProG Coder Shop Microservices** ap dung **Domain-Driven Design (DDD)** ket hop voi **Clean Architecture**, **CQRS**, va **Event-Driven Architecture** de xay dung mot nen tang thuong mai dien tu co kha nang mo rong cao.

### 1.1. So do kien truc tong quat

```mermaid
graph TB
    subgraph "Client Layer"
        Admin["App.Admin (React)"]
        Store["App.Store (React)"]
    end

    subgraph "API Gateway"
        YARP["YARP Reverse Proxy"]
    end

    subgraph "Microservices"
        Catalog["Catalog Service"]
        Basket["Basket Service"]
        Order["Order Service"]
        Inventory["Inventory Service"]
        Discount["Discount Service"]
        Notification["Notification Service"]
        Search["Search Service"]
        Report["Report Service"]
        Communication["Communication Service"]
    end

    subgraph "Infrastructure"
        RabbitMQ["RabbitMQ (MassTransit)"]
        Keycloak["Keycloak (OIDC)"]
        PG["PostgreSQL"]
        MySQL["MySQL"]
        SQLServer["SQL Server"]
        MongoDB["MongoDB"]
        Redis["Redis"]
        ES["Elasticsearch"]
    end

    Admin --> YARP
    Store --> YARP
    YARP --> Catalog & Basket & Order & Inventory & Discount & Notification & Search & Report & Communication
    Catalog & Basket & Order & Inventory & Notification -.->|Integration Events| RabbitMQ
    RabbitMQ -.-> Catalog & Order & Inventory & Search & Notification & Report & Communication
```

### 1.2. Cau truc phan lop cua moi Service

Moi microservice deu tuan thu mot cau truc 3 tang nhat quan:

```mermaid
graph TB
    subgraph "Service Structure"
        API["Api Layer<br/>(Minimal APIs, Endpoints)"]
        CORE["Core Layer"]
        WORKER["Worker Layer<br/>(Background Consumer)"]
    end

    subgraph "Core Layer (Clean Architecture)"
        DOMAIN["Domain Layer<br/>(Entities, Value Objects,<br/>Domain Events, Repositories Interface)"]
        APP["Application Layer<br/>(CQRS Handlers, DTOs,<br/>Domain Event Handlers)"]
        INFRA["Infrastructure Layer<br/>(EF Core, Repositories Impl,<br/>UnitOfWork, External Services)"]
    end

    API -->|Send Commands/Queries| APP
    WORKER -->|Consume Integration Events| APP
    APP -->|Depend on Abstractions| DOMAIN
    INFRA -->|Implement Interfaces| DOMAIN
    CORE --- DOMAIN & APP & INFRA

    style DOMAIN fill:#e1f5fe,stroke:#01579b
    style APP fill:#f3e5f5,stroke:#4a148c
    style INFRA fill:#e8f5e9,stroke:#1b5e20
```

| Service | Api | Core (Domain/Application/Infrastructure) | Worker |
|---------|-----|------------------------------------------|--------|
| Catalog | Minimal APIs | Product, Category, Brand entities | StockChanged consumer |
| Basket | Minimal APIs | ShoppingCart aggregate (MongoDB) | - |
| Order | Minimal APIs | Order aggregate, OrderItem, Value Objects | BasketCheckout, ReservationExpired consumers |
| Inventory | Minimal APIs | InventoryItem, Location, Reservation | OrderCreated/Cancelled/Delivered consumers |
| Discount | Minimal APIs | Coupon entity | - |
| Notification | Minimal APIs | Notification, Template, Delivery | UpsertedProduct consumer |
| Search | Minimal APIs | ProductEntity (Elasticsearch) | UpsertedProduct, DeletedProduct consumers |
| Report | Minimal APIs | Dashboard, Charts entities | - |
| Communication | Minimal APIs | - | OrderCreated consumer (SignalR) |

---

## 2. Domain Layer - Tactical Design Patterns

### 2.1. Entity Base Class

Moi entity trong he thong deu ke thua tu `Entity<T>` - mot abstract class cung cap:
- **Identity** (`Id`) voi kieu generic
- **Audit trail** (`CreatedOnUtc`, `CreatedBy`, `LastModifiedOnUtc`, `LastModifiedBy`)

```mermaid
classDiagram
    class IEntityId~T~ {
        <<interface>>
        +T Id
    }
    class IAuditable {
        <<interface>>
        +DateTimeOffset CreatedOnUtc
        +string CreatedBy
        +DateTimeOffset LastModifiedOnUtc
        +string LastModifiedBy
    }
    class Entity~T~ {
        <<abstract>>
        +T Id
        +DateTimeOffset CreatedOnUtc
        +string CreatedBy
        +DateTimeOffset LastModifiedOnUtc
        +string LastModifiedBy
    }
    class IAggregate~T~ {
        <<interface>>
        +IReadOnlyList~IDomainEvent~ DomainEvents
        +AddDomainEvent(IDomainEvent)
        +IDomainEvent[] ClearDomainEvents()
    }
    class Aggregate~TId~ {
        <<abstract>>
        -List~IDomainEvent~ _domainEvents
        +IReadOnlyList~IDomainEvent~ DomainEvents
        +AddDomainEvent(IDomainEvent)
        +IDomainEvent[] ClearDomainEvents()
    }

    Entity~T~ ..|> IEntityId~T~
    Entity~T~ ..|> IAuditable
    Aggregate~TId~ --|> Entity~TId~
    Aggregate~TId~ ..|> IAggregate~TId~
```

> [!NOTE]
> Moi service **tu dinh nghia** bo abstractions cua rieng minh (`Entity`, `Aggregate`, `EntityId`, interfaces). Dieu nay phu hop voi nguyen tac **Bounded Context** cua DDD - moi context co model rieng, khong chia se domain model.

### 2.2. Aggregate Roots

Aggregate Root la diem vao duy nhat de thao tac voi mot nhom entities lien quan. Trong du an:

| Service | Aggregate Root | Child Entities | Mo ta |
|---------|---------------|----------------|-------|
| **Order** | `OrderEntity` | `OrderItemEntity` | Quan ly don hang va cac san pham trong don |
| **Basket** | `ShoppingCartEntity` | `ShoppingCartItemEntity` | Gio hang va cac san pham trong gio |
| **Inventory** | `InventoryItemEntity` | `InventoryHistoryEntity`, `InventoryReservationEntity` | Ton kho, lich su va dat truoc |
| **Catalog** | Product (implicit) | `ProductImageEntity` | San pham va hinh anh |
| **Notification** | `NotificationEntity` | `DeliveryEntity`, `TemplateEntity`, `MessagePayloadEntity` | Thong bao va viec gui |

### 2.3. Value Objects

Value Objects la cac doi tuong bat bien, duoc xac dinh boi gia tri cua chung thay vi identity:

| Value Object | Service | Thuoc tinh chinh | Dac diem |
|-------------|---------|-----------------|---------|
| `Address` | Order | AddressLine, City, Country, PostalCode... | Factory method `Of()` voi validation |
| `Customer` | Order | Thong tin khach hang | Embedded trong Order |
| `OrderNo` | Order | So don hang | Dinh danh nghiep vu |
| `Discount` | Order | DiscountAmount | Ap dung giam gia |
| `Product` | Order, Inventory | Id, Price, ProductName | Reference den Catalog |
| `Money` | Catalog, Basket | Gia tri tien te | Dung chung cho gia san pham |

```mermaid
flowchart LR
    subgraph "Order Aggregate"
        OE["OrderEntity<br/>(Aggregate Root)"]
        OI["OrderItemEntity"]
        AV["Address (VO)"]
        CV["Customer (VO)"]
        DV["Discount (VO)"]
        ONV["OrderNo (VO)"]
        PV["Product (VO)"]
    end

    OE -->|"contains 1..*"| OI
    OE -->|"has"| AV
    OE -->|"has"| CV
    OE -->|"has"| DV
    OE -->|"has"| ONV
    OI -->|"references"| PV

    style OE fill:#ff9800,color:#fff
    style AV fill:#e3f2fd
    style CV fill:#e3f2fd
    style DV fill:#e3f2fd
    style ONV fill:#e3f2fd
    style PV fill:#e3f2fd
```

### 2.4. Domain Events

Domain Events duoc phat ra khi co thay doi trang thai quan trong trong Aggregate. Chung implement interface `IDomainEvent` (ke thua `INotification` cua MediatR).

| Service | Domain Event | Trigger |
|---------|-------------|---------|
| **Order** | `OrderCreatedDomainEvent` | Khi don hang duoc tao |
| **Order** | `OrderCancelledDomainEvent` | Khi don hang bi huy |
| **Order** | `OrderDeliveredDomainEvent` | Khi don hang duoc giao |
| **Basket** | `BasketCheckoutDomainEvent` | Khi khach hang checkout |
| **Catalog** | `UpsertedProductDomainEvent` | Khi san pham duoc tao/cap nhat |
| **Catalog** | `DeletedUnPublishedProductDomainEvent` | Khi san pham unpublished bi xoa |
| **Inventory** | `StockChangedDomainEvent` | Khi so luong ton kho thay doi |
| **Inventory** | `ReservationCreatedDomainEvent` | Khi dat truoc ton kho |
| **Inventory** | `ReservationCommittedDomainEvent` | Khi dat truoc duoc xac nhan |
| **Inventory** | `ReservationReleasedDomainEvent` | Khi dat truoc duoc giai phong |
| **Inventory** | `ReservationExpiredDomainEvent` | Khi dat truoc het han |
| **Inventory** | `InventoryItemDeletedDomainEvent` | Khi ton kho bi xoa |
| **Inventory** | `LocationChangedDomainEvent` | Khi dia diem ton kho thay doi |
| **Inventory** | `TransferOutDomainEvent` | Khi chuyen ton kho ra |

---

## 3. Application Layer - CQRS Pattern

### 3.1. CQRS Interfaces

He thong su dung **MediatR** lam pipeline de tach biet luong doc (Query) va ghi (Command):

```mermaid
classDiagram
    class IRequest~TResponse~ {
        <<MediatR>>
    }
    class ICommand {
        <<interface>>
    }
    class ICommand~TResponse~ {
        <<interface>>
    }
    class IQuery~TResponse~ {
        <<interface>>
    }
    class ICommandHandler~TCommand~ {
        <<interface>>
    }
    class IQueryHandler~TQuery_TResponse~ {
        <<interface>>
    }

    ICommand --|> ICommand~Unit~ : extends
    ICommand~TResponse~ --|> IRequest~TResponse~ : extends
    IQuery~TResponse~ --|> IRequest~TResponse~ : extends
```

### 3.2. Luong xu ly Command

```mermaid
flowchart TD
    A["API Endpoint<br/>(Minimal API)"] -->|"Send Command"| B["MediatR Pipeline"]
    B --> C["ValidationBehavior<br/>(FluentValidation)"]
    C --> D["LoggingBehavior<br/>(Logging Request/Response)"]
    D --> E["CommandHandler"]
    E --> F{"Xu ly nghiep vu"}
    F -->|"Thao tac Entity"| G["Aggregate Root<br/>(AddDomainEvent)"]
    G --> H["UnitOfWork.SaveChangesAsync()"]
    H --> I["DispatchDomainEventsInterceptor<br/>(EF Core Interceptor)"]
    I --> J["Domain Event Handler<br/>(INotificationHandler)"]
    J --> K["Push to OutboxMessage"]
    K --> L["SaveChanges (Atomically)"]

    style A fill:#fff3e0
    style E fill:#f3e5f5
    style G fill:#e1f5fe
    style I fill:#e8f5e9
    style J fill:#fce4ec
```

### 3.3. Danh sach Handlers theo Service

#### Catalog Service
| Loai | Handler | Chuc nang |
|------|---------|----------|
| Command | `CreateProductCommandHandler` | Tao san pham moi |
| Command | `UpdateProductCommandHandler` | Cap nhat san pham |
| Command | `DeleteProductCommandHandler` | Xoa san pham |
| Command | `PublishProductCommandHandler` | Xuat ban san pham |
| Command | `UnpublishProductCommandHandler` | Huy xuat ban |
| Command | `ChangeProductStatusCommandHandler` | Doi trang thai san pham |
| Command | `CreateCategoryCommandHandler` | Tao danh muc |
| Command | `UpdateCategoryCommandHandler` | Cap nhat danh muc |
| Command | `DeleteCategoryCommandHandler` | Xoa danh muc |
| Command | `DeleteBrandCommandHandler` | Xoa thuong hieu |
| Command | `InitialDataCommandHandler` | Khoi tao du lieu he thong |
| Query | `GetProductsQueryHandler` | Lay danh sach san pham (paging) |
| Query | `GetProductByIdQueryHandler` | Lay chi tiet san pham |
| Query | `GetPublishProductsQueryHandler` | Lay san pham da xuat ban |
| Query | `GetPublishProductByIdQueryHandler` | Lay chi tiet san pham da xuat ban |
| Query | `GetAllProductsQueryHandler` | Lay tat ca san pham |
| Query | `GetAllAvailableProductsQueryHandler` | Lay tat ca san pham hien co |
| Query | `GetCountProductQueryHandler` | Dem so luong san pham |
| Query | `GetTreeCategoriesQueryHandler` | Lay cay danh muc |
| Query | `GetAllCategoriesQueryHandler` | Lay tat ca danh muc |
| Domain Event | `UpsertedProductDomainEventHandler` | Xu ly khi san pham duoc tao/cap nhat |
| Domain Event | `DeletedUnPublishedProductDomainEventHandler` | Xu ly khi san pham unpublished bi xoa |

#### Order Service
| Loai | Handler | Chuc nang |
|------|---------|----------|
| Domain Event | `OrderCreatedDomainEventHandler` | Chuyen domain event thanh outbox message |
| Domain Event | `OrderCancelledDomainEventHandler` | Xu ly khi don hang bi huy |
| Domain Event | `OrderDeliveredDomainEventHandler` | Xu ly khi don hang duoc giao |

#### Basket Service
| Loai | Handler | Chuc nang |
|------|---------|----------|
| Command | `StoreBasketCommandHandler` | Luu gio hang |
| Command | `BasketCheckoutCommandHandler` | Checkout gio hang |
| Command | `DeleteBasketCommandHandler` | Xoa gio hang |
| Query | `GetBasketQueryHandler` | Lay thong tin gio hang |
| Domain Event | `BasketCheckoutDomainEventHandler` | Xu ly checkout event |

#### Notification Service
| Loai | Handler | Chuc nang |
|------|---------|----------|
| Command | `CreateDeliveryCommandHandler` | Tao viec gui thong bao |
| Command | `ProcessDeliveryCommandHandler` | Xu ly viec gui thong bao |
| Command | `MarkAsReadNotificationCommandHandler` | Danh dau da doc |
| Query | `GetAllNotificationsQueryHandler` | Lay tat ca thong bao |
| Query | `GetNotificationsQueryHandler` | Lay thong bao (paging) |
| Query | `GetCountNotificationUnreadQueryHandler` | Dem thong bao chua doc |
| Query | `GetTop10NotificationsUnreadQueryHandler` | Lay 10 thong bao chua doc |
| Query | `GetDueDeliveriesQueryHandler` | Lay viec gui den han |
| Query | `GetKeycloakUsersQueryHandler` | Lay danh sach user tu Keycloak |
| Query | `GetKeycloakUsersByRoleQueryHandler` | Lay user theo role |

---

## 4. Infrastructure Layer

### 4.1. Repository va Unit of Work Pattern

```mermaid
classDiagram
    class IUnitOfWork {
        <<interface>>
        +IOrderRepository Orders
        +IOrderItemRepository OrderItems
        +IInboxMessageRepository InboxMessages
        +IOutboxMessageRepository OutboxMessages
        +SaveChangesAsync() Task~int~
        +BeginTransactionAsync() Task~IDbTransaction~
    }

    class IRepository {
        <<interface>>
    }

    class IOrderRepository {
        <<interface>>
    }

    class UnitOfWork {
        -ApplicationDbContext _context
        +IOrderRepository Orders
        +IOrderItemRepository OrderItems
        +SaveChangesAsync() Task~int~
        +BeginTransactionAsync() Task~IDbTransaction~
    }

    IUnitOfWork <|.. UnitOfWork
    IUnitOfWork --> IOrderRepository
    IUnitOfWork --> IOrderItemRepository
    IUnitOfWork --> IOutboxMessageRepository
    IUnitOfWork --> IInboxMessageRepository
    IOrderRepository --|> IRepository
```

### 4.2. Repositories theo Service

| Service | Command Repository | Query Repository | Dac biet |
|---------|-------------------|-----------------|---------|
| **Order** | `IOrderRepository`, `IOrderItemRepository` | (chung) | `IOutboxMessageRepository`, `IInboxMessageRepository` |
| **Inventory** | `IInventoryItemRepository`, `ILocationRepository`, `IInventoryReservationRepository`, `IInventoryHistoryRepository` | (chung) | `IOutboxMessageRepository`, `IInboxMessageRepository` |
| **Basket** | `IBasketRepository` | (chung) | `IOutboxRepository` (MongoDB) |
| **Catalog** | (implicit qua DbContext) | (implicit) | `IOutboxRepository` |
| **Discount** | `ICouponRepository` | (chung) | - |
| **Notification** | `ICommandNotificationRepository`, `ICommandDeliveryRepository` | `IQueryNotificationRepository`, `IQueryDeliveryRepository`, `IQueryTemplateRepository` | Tach Command/Query Repository |
| **Search** | `IProductRepository` | (chung) | Elasticsearch-based |
| **Report** | `IDashboardTotalRepository`, `IOrderGrowthLineChartRepository`, `ITopProductPieChartRepository` | (chung) | - |

> [!IMPORTANT]
> **Notification Service** tach biet ro rang giua Command Repository va Query Repository, the hien CQRS o tang data access.

### 4.3. DispatchDomainEventsInterceptor

Day la EF Core `SaveChangesInterceptor` duoc su dung trong **Order** va **Inventory** services. No tu dong dispatch domain events khi `SaveChangesAsync` duoc goi:

```mermaid
sequenceDiagram
    participant Handler as CommandHandler
    participant UoW as UnitOfWork
    participant DbCtx as DbContext
    participant Interceptor as DispatchDomainEventsInterceptor
    participant MediatR as MediatR (Publish)
    participant DomainHandler as DomainEventHandler

    Handler->>UoW: SaveChangesAsync()
    UoW->>DbCtx: SaveChangesAsync()
    DbCtx->>Interceptor: SavingChangesAsync()
    Interceptor->>DbCtx: Get entities with DomainEvents
    loop For each DomainEvent
        Interceptor->>MediatR: Publish(domainEvent)
        MediatR->>DomainHandler: Handle(domainEvent)
        DomainHandler->>UoW: Push to OutboxMessage
    end
    Interceptor->>DbCtx: ClearDomainEvents()
    DbCtx-->>Handler: SaveChanges completed
```

---

## 5. Event-Driven Architecture - Giao tiep giua cac Service

### 5.1. Integration Events

Integration Events la co che giao tiep **bat dong bo** giua cac Bounded Context thong qua RabbitMQ (MassTransit). Tat ca deu ke thua tu `IntegrationEvent` record trong Shared/EventSourcing:

| Integration Event | Publisher | Consumers |
|------------------|-----------|-----------|
| `BasketCheckoutIntegrationEvent` | Basket | **Order** (tao don hang) |
| `OrderCreatedIntegrationEvent` | Order | **Inventory** (dat truoc ton kho), **Communication** (realtime notification), **Report** (cap nhat bieu do) |
| `OrderCancelledIntegrationEvent` | Order | **Inventory** (giai phong ton kho) |
| `OrderDeliveredIntegrationEvent` | Order | **Inventory** (xac nhan dat truoc) |
| `UpsertedProductIntegrationEvent` | Catalog | **Search** (cap nhat index), **Notification** (thong bao san pham moi) |
| `DeletedUnPublishedProductIntegrationEvent` | Catalog | **Search** (xoa khoi index) |
| `StockChangedIntegrationEvent` | Inventory | **Catalog** (cap nhat trang thai ton kho) |
| `ReservationExpiredIntegrationEvent` | Inventory | **Order** (huy don hang het han) |

### 5.2. Outbox Pattern

De dam bao consistency giua viec luu du lieu va phat su kien, he thong su dung **Outbox Pattern**:

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Database
    participant Outbox as OutboxMessage Table
    participant Worker as Background Worker
    participant MQ as RabbitMQ (MassTransit)

    App->>DB: Begin Transaction
    App->>DB: Save Entity Changes
    App->>Outbox: Insert OutboxMessage (Event serialized as JSON)
    App->>DB: Commit Transaction
    Note over App,DB: Atomicity dam bao!

    loop Polling interval
        Worker->>Outbox: Query unprocessed messages
        Outbox-->>Worker: Return messages
        Worker->>MQ: Publish Integration Event
        Worker->>Outbox: Mark as processed
    end
```

### 5.3. Inbox Pattern

**Order** va **Inventory** services su dung **Inbox Pattern** de dam bao idempotency khi nhan Integration Events:

```mermaid
sequenceDiagram
    participant MQ as RabbitMQ
    participant Consumer as Integration Event Consumer
    participant Inbox as InboxMessage Table
    participant App as Application Logic

    MQ->>Consumer: Deliver message
    Consumer->>Inbox: Check message ID exists?
    alt Already processed
        Inbox-->>Consumer: Found (duplicate)
        Consumer-->>MQ: ACK (skip)
    else New message
        Inbox-->>Consumer: Not found
        Consumer->>App: Process business logic
        Consumer->>Inbox: Insert message ID
        Consumer-->>MQ: ACK
    end
```

---

## 6. Luong nghiep vu chinh

### 6.1. Luong Dat Hang (Order Flow)

Day la luong nghiep vu phuc tap nhat, bao gom nhieu Bounded Context tuong tac voi nhau theo mo hinh **Choreography-based Saga**:

```mermaid
sequenceDiagram
    participant Client
    participant BasketAPI as Basket API
    participant BasketDomain as Basket Domain
    participant RabbitMQ
    participant OrderWorker as Order Worker
    participant OrderDomain as Order Domain
    participant InventoryWorker as Inventory Worker
    participant InventoryDomain as Inventory Domain
    participant CommWorker as Communication Worker
    participant ReportSvc as Report Service

    Client->>BasketAPI: POST /basket/checkout
    BasketAPI->>BasketDomain: BasketCheckoutCommand (MediatR)
    BasketDomain->>BasketDomain: AddDomainEvent(BasketCheckoutDomainEvent)
    BasketDomain->>BasketDomain: DomainEventHandler -> Push to Outbox
    Note over BasketDomain: Outbox Worker publish event

    BasketDomain--)RabbitMQ: BasketCheckoutIntegrationEvent

    RabbitMQ--)OrderWorker: Consume
    OrderWorker->>OrderDomain: Create OrderEntity
    OrderDomain->>OrderDomain: AddDomainEvent(OrderCreatedDomainEvent)
    OrderDomain->>OrderDomain: DomainEventHandler -> Push to Outbox
    Note over OrderDomain: Outbox Worker publish event

    OrderDomain--)RabbitMQ: OrderCreatedIntegrationEvent

    par Inventory Processing
        RabbitMQ--)InventoryWorker: Consume
        InventoryWorker->>InventoryDomain: Create InventoryReservation
        InventoryDomain->>InventoryDomain: AddDomainEvent(ReservationCreatedDomainEvent)
    and Realtime Notification
        RabbitMQ--)CommWorker: Consume
        CommWorker->>CommWorker: Send via SignalR
    and Report Update
        RabbitMQ--)ReportSvc: Consume
        ReportSvc->>ReportSvc: Update Dashboard
    end
```

### 6.2. Luong Huy Don Hang (Compensation)

```mermaid
sequenceDiagram
    participant Admin
    participant OrderAPI as Order API
    participant OrderDomain as Order Domain
    participant RabbitMQ
    participant InventoryWorker as Inventory Worker
    participant InventoryDomain as Inventory Domain

    Admin->>OrderAPI: Cancel Order (reason)
    OrderAPI->>OrderDomain: CancelOrder(reason, performBy)
    OrderDomain->>OrderDomain: UpdateStatus(Canceled)
    OrderDomain->>OrderDomain: AddDomainEvent(OrderCancelledDomainEvent)
    OrderDomain->>OrderDomain: DomainEventHandler -> Push to Outbox

    OrderDomain--)RabbitMQ: OrderCancelledIntegrationEvent

    RabbitMQ--)InventoryWorker: Consume
    InventoryWorker->>InventoryDomain: Release Reservation
    InventoryDomain->>InventoryDomain: AddDomainEvent(ReservationReleasedDomainEvent)
    InventoryDomain->>InventoryDomain: Restore Stock
```

### 6.3. Luong Catalog -> Search Sync

```mermaid
sequenceDiagram
    participant Admin
    participant CatalogAPI as Catalog API
    participant CatalogDomain as Catalog Domain
    participant RabbitMQ
    participant SearchWorker as Search Worker
    participant SearchApp as Search Application
    participant ES as Elasticsearch

    Admin->>CatalogAPI: Create/Update Product
    CatalogAPI->>CatalogDomain: CreateProductCommand (MediatR)
    CatalogDomain->>CatalogDomain: AddDomainEvent(UpsertedProductDomainEvent)
    CatalogDomain->>CatalogDomain: DomainEventHandler -> Push to Outbox

    CatalogDomain--)RabbitMQ: UpsertedProductIntegrationEvent

    RabbitMQ--)SearchWorker: Consume
    SearchWorker->>SearchApp: UpsertProductCommand (MediatR)
    SearchApp->>ES: Index/Update Document
```

---

## 7. Cross-Cutting Concerns

### 7.1. MediatR Pipeline Behaviors

```mermaid
flowchart LR
    Request["Request"] --> VB["ValidationBehavior<br/>(FluentValidation)"]
    VB --> LB["LoggingBehavior<br/>(Structured Logging)"]
    LB --> Handler["Handler<br/>(Business Logic)"]
    Handler --> Response["Response"]

    style VB fill:#ffcdd2
    style LB fill:#c8e6c9
```

| Behavior | Chuc nang |
|----------|----------|
| `ValidationBehavior` | Tu dong validate request bang FluentValidation truoc khi den handler |
| `LoggingBehavior` | Log thong tin request/response de tracing va debug |

### 7.2. Shared Libraries

| Library | Chuc nang |
|---------|----------|
| **BuildingBlocks** | CQRS interfaces, Behaviors, Exceptions, Pagination, Swagger, Authentication, Logging, Validators |
| **Common** | Configurations, Helpers, Models dung chung |
| **Contracts** | Shared DTOs va contracts giua cac service |
| **EventSourcing** | Integration Events definitions, MassTransit extensions |

### 7.3. Exception Handling

```mermaid
classDiagram
    class Exception {
        <<System>>
    }
    class DomainException {
        Domain-specific errors
    }
    class ClientValidationException {
        Input validation errors
    }
    class NotFoundException {
        Resource not found
    }
    class UnauthorizedException {
        Authentication errors
    }
    class NoPermissionException {
        Authorization errors
    }
    class InternalServerException {
        Unexpected errors
    }

    Exception <|-- DomainException
    Exception <|-- ClientValidationException
    Exception <|-- NotFoundException
    Exception <|-- UnauthorizedException
    Exception <|-- NoPermissionException
    Exception <|-- InternalServerException
```

---

## 8. Database Strategy

| Service | Database | Technology | Ly do |
|---------|----------|-----------|-------|
| Catalog | PostgreSQL | EF Core | Quan he phuc tap giua Product/Category/Brand |
| Order | PostgreSQL | EF Core | ACID transactions cho don hang |
| Inventory | PostgreSQL | EF Core | Consistency cho ton kho |
| Basket | MongoDB | MongoDB Driver | Schema linh hoat cho gio hang |
| Discount | PostgreSQL | EF Core/Dapper | Don gian, hieu suat cao |
| Notification | PostgreSQL | EF Core | Luu tru thong bao va template |
| Search | Elasticsearch | NEST Client | Full-text search |
| Report | MySQL | EF Core | Read-optimized cho bao cao |
| Communication | - | In-memory | Chi xu ly realtime, khong luu tru |

---

## 9. Bounded Context Map

```mermaid
graph LR
    subgraph "Core Domain"
        Order["Order BC"]
        Inventory["Inventory BC"]
        Catalog["Catalog BC"]
    end

    subgraph "Supporting Domain"
        Basket["Basket BC"]
        Discount["Discount BC"]
    end

    subgraph "Generic Domain"
        Notification["Notification BC"]
        Search["Search BC"]
        Report["Report BC"]
        Communication["Communication BC"]
    end

    Basket -->|"BasketCheckout<br/>(Async/Event)"| Order
    Order -->|"OrderCreated<br/>(Async/Event)"| Inventory
    Order -->|"OrderCancelled<br/>(Async/Event)"| Inventory
    Order -->|"OrderDelivered<br/>(Async/Event)"| Inventory
    Inventory -->|"ReservationExpired<br/>(Async/Event)"| Order
    Inventory -->|"StockChanged<br/>(Async/Event)"| Catalog
    Catalog -->|"ProductUpserted<br/>(Async/Event)"| Search
    Catalog -->|"ProductUpserted<br/>(Async/Event)"| Notification
    Order -->|"OrderCreated<br/>(Async/Event)"| Communication
    Order -->|"OrderCreated<br/>(Async/Event)"| Report

    style Order fill:#e65100,color:#fff
    style Inventory fill:#e65100,color:#fff
    style Catalog fill:#e65100,color:#fff
    style Basket fill:#1565c0,color:#fff
    style Discount fill:#1565c0,color:#fff
    style Notification fill:#2e7d32,color:#fff
    style Search fill:#2e7d32,color:#fff
    style Report fill:#2e7d32,color:#fff
    style Communication fill:#2e7d32,color:#fff
```

> [!TIP]
> Tat ca giao tiep giua cac Bounded Context deu thong qua **Integration Events** (bat dong bo qua RabbitMQ). Khong co Bounded Context nao goi truc tiep API cua context khac, dam bao **loose coupling** toi da.

---

## 10. Tong ket

### Diem manh cua thiet ke DDD trong du an

1. **Bounded Context ro rang**: Moi service co domain model rieng, khong chia se entity giua cac context.
2. **Rich Domain Model**: Aggregate Roots chua business logic (factory methods, domain methods, domain events) thay vi anemic model.
3. **Event-Driven Communication**: Tat ca giao tiep inter-service deu bat dong bo qua events, dao bao loose coupling.
4. **Outbox/Inbox Pattern**: Dam bao reliable messaging va idempotency.
5. **CQRS nhat quan**: Tach biet ro rang Command va Query handlers qua MediatR pipeline.
6. **Clean Architecture**: Dependency inversion - Domain layer khong phu thuoc vao Infrastructure.

### Diem can cai thien

1. **Value Objects chua immutable hoan toan**: `Address` co public setters, nen su dung `record` hoac private setters.
2. **Thieu domain validation tap trung**: Mot so entities khong co invariant checks day du trong constructor/factory methods.
3. **Bo abstractions lap lai**: Moi service dinh nghia lai `Entity`, `Aggregate`, `EntityId` - co the extract thanh shared package cho DDD base classes ma van giu duoc bounded context independence.

---

*Tai lieu nay duoc tao tu dong bang code-graph-rag-index MCP server va phan tich source code truc tiep.*
*Ngay tao: 2026-02-15*
