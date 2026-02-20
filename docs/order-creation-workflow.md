# Order Creation Workflow Documentation

Tài liệu này mô tả chi tiết workflow tạo đơn hàng trong hệ thống ProgCoder Shop Microservices sử dụng Mermaid diagrams.

---

## Mục lục (Table of Contents)

1. [Overall Architecture Diagram](#1-overall-architecture-diagram)
2. [Sequence Diagram - Order Creation Flow](#2-sequence-diagram---order-creation-flow)
3. [Flowchart - CreateOrderCommand Handler](#3-flowchart---createordercommand-handler)
4. [Outbox Pattern Flow](#4-outbox-pattern-flow)
5. [Basket Checkout to Order Flow](#5-basket-checkout-to-order-flow)
6. [Class Diagram - Order Domain Model](#6-class-diagram---order-domain-model)
7. [State Machine - Order Status Flow](#7-state-machine---order-status-flow)
8. [Component Diagram - Order Service Internals](#8-component-diagram---order-service-internals)

---

## 1. Overall Architecture Diagram

Sơ đồ tổng quan kiến trúc và các service liên quan đến việc tạo đơn hàng.

```mermaid
flowchart TB
    subgraph Client["Client Layer"]
        Web["Web App / Store"]
        Admin["Admin App"]
    end

    subgraph APIGateway["API Gateway (YARP)"]
        Gateway["Gateway Service<br/>Port: 15009"]
    end

    subgraph OrderService["Order Service"]
        OrderAPI["Order.Api<br/>Carter Endpoints"]
        OrderApp["Order.Application<br/>CQRS Handlers"]
        OrderDomain["Order.Domain<br/>Aggregates & Events"]
        OrderInfra["Order.Infrastructure<br/>Repository & gRPC"]
        OrderDB[("PostgreSQL<br/>Order DB")]
        
        subgraph OrderWorkers["Background Workers"]
            OutboxWorker["Order.Worker.Outbox<br/>Outbox Processor"]
            ConsumerWorker["Order.Worker.Consumer<br/>Event Consumer"]
        end
    end

    subgraph BasketService["Basket Service"]
        BasketAPI["Basket.Api"]
        BasketDomain["Basket.Domain"]
    end

    subgraph CatalogService["Catalog Service"]
        CatalogAPI["Catalog.Api"]
        CatalogGrpc["Catalog.Grpc"]
    end

    subgraph DiscountService["Discount Service"]
        DiscountAPI["Discount.Api"]
        DiscountGrpc["Discount.Grpc"]
    end

    subgraph InventoryService["Inventory Service"]
        InventoryAPI["Inventory.Api"]
        InventoryConsumer["Inventory.Worker.Consumer"]
    end

    subgraph CommunicationService["Communication Service"]
        CommAPI["Communication.Api"]
    end

    subgraph MessageBroker["Message Broker"]
        RabbitMQ["RabbitMQ"]
    end

    %% Direct API Flow
    Web -->|"1. POST /orders"| Gateway
    Gateway -->|"2. Route to Order.Api"| OrderAPI
    OrderAPI -->|"3. Send Command"| OrderApp
    
    %% CQRS Flow
    OrderApp -->|"4. Domain Logic"| OrderDomain
    OrderApp -->|"5. gRPC Calls"| OrderInfra
    OrderInfra -->|"6a. Get Products"| CatalogGrpc
    OrderInfra -->|"6b. Apply Coupon"| DiscountGrpc
    OrderInfra -->|"7. Save Order"| OrderDB
    
    %% Outbox Pattern
    OrderDomain -->|"8. Domain Events"| OrderApp
    OrderApp -->|"9. Save to Outbox"| OrderDB
    OrderDB -->|"10. Poll Messages"| OutboxWorker
    OutboxWorker -->|"11. Publish"| RabbitMQ
    
    %% Event Consumers
    RabbitMQ -->|"12. Consume"| InventoryConsumer
    RabbitMQ -->|"13. Consume"| CommAPI
    
    %% Basket Checkout Flow
    Web -->|"A. Checkout Basket"| BasketAPI
    BasketAPI -->|"B. Domain Events"| BasketDomain
    BasketDomain -->|"C. Publish Event"| RabbitMQ
    RabbitMQ -->|"D. Consume"| ConsumerWorker
    ConsumerWorker -->|"E. Create Order"| OrderApp

    style OrderService fill:#e1f5fe,stroke:#01579b,stroke-width:3px
    style OrderApp fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style OrderWorkers fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
```

---

## 2. Sequence Diagram - Order Creation Flow

Sơ đồ sequence chi tiết quá trình tạo đơn hàng từ API đến Database và các service liên quan.

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Client as Web/Admin App
    participant Gateway as API Gateway
    participant OrderAPI as Order.Api
    participant Mediator as IMediator
    participant Handler as CreateOrderCommandHandler
    participant UnitOfWork as IUnitOfWork
    participant OrderEntity as OrderEntity
    participant CatalogSvc as CatalogGrpcService
    participant Catalog as Catalog.Grpc
    participant DiscountSvc as DiscountGrpcService
    participant Discount as Discount.Grpc
    participant Repo as OrderRepository
    participant DB as PostgreSQL

    User->>+Client: 1. Tạo đơn hàng
    Client->>+Gateway: POST /orders
    Gateway->>+OrderAPI: Route Request
    OrderAPI->>OrderAPI: Extract Current User
    OrderAPI->>+Mediator: Send(CreateOrderCommand)
    
    Mediator->>+Handler: Handle(command)
    
    Note over Handler: Generate OrderId & OrderNo
    
    Handler->>Handler: 2. Create Customer VO
    Handler->>Handler: 3. Create Address VO
    Handler->>Handler: 4. Create OrderEntity
    
    Handler->>+CatalogSvc: GetAllAvailableProductsAsync(ids)
    CatalogSvc->>+Catalog: GetProductsAsync gRPC
    Catalog-->>-CatalogSvc: ProductResponse
    CatalogSvc-->>-Handler: List<ProductDto>
    
    loop Validate & Add Items
        Handler->>Handler: Validate Product Exists
        Handler->>Handler: Validate Quantity > 0
        Handler->>Handler: Create Product VO
        Handler->>+OrderEntity: AddOrderItem(product, qty)
        OrderEntity->>OrderEntity: Add to OrderItems list
        OrderEntity-->>-Handler: OrderItemEntity
    end
    
    alt Has CouponCode
        Handler->>Handler: Calculate TotalAmount
        Handler->>+DiscountSvc: EvaluateCouponAsync(code, amount)
        DiscountSvc->>+Discount: EvaluateCouponAsync gRPC
        Discount-->>-DiscountSvc: EvaluateCouponResponse
        DiscountSvc-->>-Handler: EvaluateCouponResponse
        
        Handler->>+DiscountSvc: ApplyCouponAsync(code, amount)
        DiscountSvc->>+Discount: ApplyCouponAsync gRPC
        Discount-->>-DiscountSvc: ApplyCouponResponse
        DiscountSvc-->>-Handler: ApplyCouponResponse
        
        Handler->>Handler: Create Discount VO
        Handler->>+OrderEntity: ApplyDiscount(discount)
        OrderEntity->>OrderEntity: Set Discount property
        OrderEntity-->>-Handler: void
    end
    
    Handler->>+OrderEntity: OrderCreated()
    OrderEntity->>OrderEntity: Add Domain Event
    Note right of OrderEntity: OrderCreatedDomainEvent
    OrderEntity-->>-Handler: void
    
    Handler->>+UnitOfWork: Orders.AddAsync(order)
    UnitOfWork->>+Repo: AddAsync(order)
    Repo-->>-UnitOfWork: void
    UnitOfWork-->>-Handler: void
    
    Handler->>+UnitOfWork: SaveChangesAsync()
    UnitOfWork->>+DB: INSERT Order + OrderItems
    DB-->>-UnitOfWork: Success
    UnitOfWork-->>-Handler: int (rows affected)
    
    Handler-->>-Mediator: OrderId (Guid)
    Mediator-->>-OrderAPI: OrderId
    OrderAPI-->>-Gateway: 201 Created + OrderId
    Gateway-->>-Client: 201 Created + OrderId
    Client-->>-User: Hiển thị thành công
```

---

## 3. Flowchart - CreateOrderCommand Handler

Sơ đồ luồng chi tiết xử lý trong CreateOrderCommandHandler.

```mermaid
flowchart TD
    Start([Start]) --> ValidateInput["Validate Input<br/>- Customer Info<br/>- Shipping Address<br/>- Order Items"]
    
    ValidateInput --> CheckInput{"Input Valid?"}
    CheckInput -->|No| ThrowValidation["Throw ValidationException"]
    ThrowValidation --> EndFail([End - Error])
    
    CheckInput -->|Yes| GenIds["Generate OrderId<br/>Generate OrderNo"]
    
    GenIds --> CreateVO["Create Value Objects<br/>- Customer<br/>- Address"]
    
    CreateVO --> CreateEntity["Create OrderEntity<br/>via OrderEntity.Create()"]
    
    CreateEntity --> CallCatalog["Call CatalogGrpcService<br/>GetAllAvailableProductsAsync"]
    
    CallCatalog --> CatalogResponse{"Catalog<br/>Response?"}
    CatalogResponse -->|Null/Error| LogCatalog["Log Warning"]
    LogCatalog --> ThrowNotFound["Throw NotFoundException<br/>ProductNotFound"]
    ThrowNotFound --> EndFail
    
    CatalogResponse -->|Success| InitCounter["Initialize Item Counter<br/>i = 0"]
    
    InitCounter --> LoopCondition{"i &lt; OrderItems.Count?"}
    LoopCondition -->|No| CheckCoupon{"Has<br/>CouponCode?"}
    
    LoopCondition -->|Yes| GetItem["Get OrderItem Dto[i]"]
    GetItem --> FindProduct["Find Product in<br/>Catalog Response"]
    
    FindProduct --> ProductFound{"Product<br/>Found?"}
    ProductFound -->|No| LogProduct["Log Warning"]
    LogProduct --> ThrowProdNotFound["Throw NotFoundException<br/>ProductNotFound"]
    ThrowProdNotFound --> EndFail
    
    ProductFound -->|Yes| CheckQty{"Quantity &gt; 0?"}
    CheckQty -->|No| ThrowQty["Throw ValidationException<br/>ProductQuantityInvalid"]
    ThrowQty --> EndFail
    
    CheckQty -->|Yes| CreateProductVO["Create Product VO"]
    CreateProductVO --> AddItem["OrderEntity.AddOrderItem<br/>product, quantity"]
    AddItem --> IncrementCounter["i++"]
    IncrementCounter --> LoopCondition
    
    CheckCoupon -->|Yes| CalcTotal["Calculate TotalAmount"]
    CalcTotal --> EvalCoupon["Call DiscountGrpc<br/>EvaluateCouponAsync"]
    
    EvalCoupon --> EvalResponse{"Response?"}
    EvalResponse -->|Null| LogEval["Log Warning"]
    LogEval --> ThrowCoupon["Throw NotFoundException<br/>CouponNotFound"]
    ThrowCoupon --> EndFail
    
    EvalResponse -->|Success| ApplyCoupon["Call DiscountGrpc<br/>ApplyCouponAsync"]
    ApplyCoupon --> ApplyResponse{"Response?"}
    ApplyResponse -->|Null| LogApply["Log Warning"]
    LogApply --> ThrowApply["Throw NotFoundException<br/>CouponNotFound"]
    ThrowApply --> EndFail
    
    ApplyResponse -->|Success| CreateDiscountVO["Create Discount VO"]
    CreateDiscountVO --> ApplyDiscount["OrderEntity.ApplyDiscount"]
    ApplyDiscount --> OrderCreated
    
    CheckCoupon -->|No| OrderCreated["OrderEntity.OrderCreated()"]
    
    OrderCreated --> RaiseEvent["Raise Domain Event<br/>OrderCreatedDomainEvent"]
    RaiseEvent --> SaveOrder["unitOfWork.Orders<br/>.AddAsync(order)"]
    SaveOrder --> SaveChanges["unitOfWork<br/>.SaveChangesAsync()"]
    
    SaveChanges --> SaveSuccess{"Save<br/>Success?"}
    SaveSuccess -->|No| ThrowSave["Throw Exception"]
    ThrowSave --> EndFail
    
    SaveSuccess -->|Yes| ReturnId["Return OrderId"]
    ReturnId --> EndSuccess([End - Success])

    style Start fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style EndSuccess fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style EndFail fill:#ffebee,stroke:#c62828,stroke-width:2px
    style ThrowValidation fill:#ffebee,stroke:#c62828
    style ThrowNotFound fill:#ffebee,stroke:#c62828
    style ThrowProdNotFound fill:#ffebee,stroke:#c62828
    style ThrowQty fill:#ffebee,stroke:#c62828
    style ThrowCoupon fill:#ffebee,stroke:#c62828
    style ThrowApply fill:#ffebee,stroke:#c62828
    style ThrowSave fill:#ffebee,stroke:#c62828
    style OrderCreated fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style RaiseEvent fill:#fff3e0,stroke:#e65100,stroke-width:2px
```

---

## 4. Outbox Pattern Flow

Sơ đồ luồng xử lý Outbox Pattern để đảm bảo reliable message publishing.

```mermaid
flowchart TB
    subgraph DomainLayer["Domain Layer"]
        DomainEvent["OrderCreatedDomainEvent<br/>raised by OrderEntity"]
    end

    subgraph ApplicationLayer["Application Layer"]
        EventHandler["OrderCreatedDomainEventHandler<br/>(INotificationHandler)"]
        CreateOutbox["Create OutboxMessageEntity"]
        Serialize["Serialize Event to JSON"]
    end

    subgraph InfrastructureLayer["Infrastructure Layer"]
        UoW["UnitOfWork"]
        OutboxRepo["OutboxMessageRepository"]
        OrderRepo["OrderRepository"]
    end

    subgraph Database["Database (PostgreSQL)"]
        OrdersTable["Orders Table"]
        OrderItemsTable["OrderItems Table"]
        OutboxTable["OutboxMessages Table<br/>- Id<br/>- Type<br/>- Content<br/>- ProcessedOnUtc<br/>- AttemptCount<br/>- ErrorMessage"]
    end

    subgraph OutboxWorker["Outbox Worker Background Service"]
        Poll["Poll Unprocessed<br/>Messages (Batch)"]
        Deserialize["Deserialize Message<br/>Type Cache Lookup"]
        Publish["Publish via<br/>MassTransit IPublishEndpoint"]
        UpdateSuccess["Update ProcessedOnUtc"]
        UpdateError["Update ErrorMessage<br/>Increment AttemptCount"]
        RetryLogic["Calculate NextAttemptOnUtc<br/>Exponential Backoff + Jitter"]
    end

    subgraph MessageBroker["Message Broker"]
        RabbitMQ["RabbitMQ"]
    end

    subgraph Consumers["Event Consumers"]
        InventoryHandler["Inventory Service<br/>OrderCreatedIntegrationEventHandler"]
        CommHandler["Communication Service<br/>OrderCreatedIntegrationEventHandler"]
    end

    %% Transactional Outbox Flow
    DomainEvent -->|"1. MediatR Notification"| EventHandler
    EventHandler -->|"2. Convert to IntegrationEvent"| CreateOutbox
    CreateOutbox --> Serialize
    Serialize -->|"3. Add to UnitOfWork"| UoW
    
    %% Same Transaction
    UoW -->|"4a. Save Order"| OrderRepo
    UoW -->|"4b. Save Outbox"| OutboxRepo
    OrderRepo -->|"5a. INSERT"| OrdersTable
    OrderRepo -->|"5b. INSERT"| OrderItemsTable
    OutboxRepo -->|"5c. INSERT"| OutboxTable
    
    %% Outbox Processor
    OutboxTable -->|"6. Poll every X seconds"| Poll
    Poll -->|"7. Get unprocessed batch"| Deserialize
    Deserialize -->|"8. Type resolution"| Publish
    Publish -->|"9. Publish"| RabbitMQ
    
    Publish -->|"10a. Success"| UpdateSuccess
    Publish -->|"10b. Failure"| UpdateError
    UpdateError --> RetryLogic
    RetryLogic -->|"Retry later"| OutboxTable
    UpdateSuccess -->|"Mark processed"| OutboxTable
    
    %% Message Consumption
    RabbitMQ -->|"11a. Route"| InventoryHandler
    RabbitMQ -->|"11b. Route"| CommHandler

    style DomainLayer fill:#e3f2fd,stroke:#1565c0
    style ApplicationLayer fill:#e8f5e9,stroke:#2e7d32
    style InfrastructureLayer fill:#fff3e0,stroke:#e65100
    style Database fill:#f3e5f5,stroke:#4a148c
    style OutboxWorker fill:#fce4ec,stroke:#880e4f
    style MessageBroker fill:#e0f2f1,stroke:#00695c
    style Consumers fill:#e8eaf6,stroke:#3f51b5
```

---

## 5. Basket Checkout to Order Flow

Sơ đồ luồng tạo đơn hàng từ Basket Checkout thông qua Integration Events.

```mermaid
flowchart TB
    subgraph Client["Client"]
        Web["Web/Store App"]
    end

    subgraph BasketService["Basket Service"]
        BasketAPI["Basket.Api"]
        BasketHandler["CheckoutBasketCommandHandler"]
        BasketDomain["BasketCheckoutDomainEvent"]
        BasketEventHandler["BasketCheckoutDomainEventHandler"]
    end

    subgraph OrderService["Order Service"]
        ConsumerWorker["Order.Worker.Consumer<br/>BasketCheckoutIntegrationEventHandler"]
        InboxRepo["InboxMessageRepository"]
        Mediator["IMediator"]
        CreateOrderCmd["CreateOrderCommand"]
        CreateOrderHandler["CreateOrderCommandHandler"]
    end

    subgraph MessageBroker["Message Broker"]
        RabbitMQ["RabbitMQ<br/>MassTransit"]
    end

    subgraph OtherServices["Other Services"]
        Catalog["Catalog Service"]
        Discount["Discount Service"]
        DB[("Order DB")]
    end

    %% Flow
    Web -->|"1. POST /basket/checkout"| BasketAPI
    BasketAPI -->|"2. Send Command"| BasketHandler
    BasketHandler -->|"3. Business Logic"| BasketDomain
    BasketDomain -->|"4. Domain Event Raised"| BasketEventHandler
    
    BasketEventHandler -->|"5. Publish Integration Event"| RabbitMQ
    
    RabbitMQ -->|"6. Consume Event"| ConsumerWorker
    
    ConsumerWorker -->|"7a. Check Idempotency<br/>Message already processed?"| InboxRepo
    InboxRepo -->|"Already processed"| EndSkip([Skip Processing])
    
    ConsumerWorker -->|"7b. Save to Inbox<br/>Status: Processing"| InboxRepo
    
    ConsumerWorker -->|"8. Map DTO"| CreateOrderCmd
    CreateOrderCmd -->|"9. Send Command"| Mediator
    Mediator -->|"10. Handle"| CreateOrderHandler
    
    CreateOrderHandler -->|"11a. Get Products"| Catalog
    CreateOrderHandler -->|"11b. Apply Coupon"| Discount
    CreateOrderHandler -->|"12. Save Order"| DB
    
    CreateOrderHandler -->|"13. Success"| ConsumerWorker
    ConsumerWorker -->|"14. Update Inbox<br/>Status: Completed"| InboxRepo
    
    CreateOrderHandler -.->|"Error"| ConsumerWorker
    ConsumerWorker -->|"Update Inbox<br/>Status: Error"| InboxRepo

    style Client fill:#e3f2fd,stroke:#1565c0
    style BasketService fill:#e8f5e9,stroke:#2e7d32
    style OrderService fill:#fff3e0,stroke:#e65100
    style MessageBroker fill:#f3e5f5,stroke:#4a148c
    style OtherServices fill:#e0f2f1,stroke:#00695c
```

---

## 6. Class Diagram - Order Domain Model

Sơ đồ lớp cho Order Domain Model và các thành phần liên quan.

```mermaid
classDiagram
    direction TB

    class OrderEntity {
        +Guid Id
        +List~OrderItemEntity~ OrderItems
        +Customer Customer
        +OrderNo OrderNo
        +Address ShippingAddress
        +OrderStatus Status
        +Discount Discount
        +string Notes
        +decimal TotalPrice
        +decimal FinalPrice
        +Create()$ OrderEntity
        +UpdateShippingAddress(address)
        +UpdateCustomerInfo(customer)
        +AddOrderItem(product, quantity)
        +RemoveOrderItem(orderItemId)
        +ApplyDiscount(discount)
        +UpdateStatus(status)
        +CancelOrder(reason)
        +RefundOrder(reason)
        +OrderCreated()
        +OrderDelivered()
    }

    class OrderItemEntity {
        +Guid Id
        +Guid OrderId
        +Product Product
        +int Quantity
        +decimal UnitPrice
        +decimal LineTotal
        +Create(orderId, product, quantity, price)$ OrderItemEntity
    }

    class Customer {
        +string FirstName
        +string LastName
        +string Email
        +string Phone
        +string FullName
        +Create(firstName, lastName, email, phone)$ Customer
    }

    class Address {
        +string Street
        +string City
        +string State
        +string Country
        +string ZipCode
        +Create(street, city, state, country, zipCode)$ Address
    }

    class Product {
        +Guid Id
        +string Name
        +string Sku
        +decimal Price
        +Create(id, name, sku, price)$ Product
    }

    class OrderNo {
        +string Value
        +Generate()$ OrderNo
    }

    class Discount {
        +string Code
        +decimal Amount
        +decimal Percentage
        +Create(code, amount, percentage)$ Discount
    }

    class OrderStatus {
        <<enumeration>>
        Pending
        Processing
        Paid
        Shipped
        Delivered
        Cancelled
        Refunded
    }

    class Aggregate~T~ {
        <<abstract>>
        +T Id
        +List~IDomainEvent~ DomainEvents
        +AddDomainEvent(event)
        +ClearDomainEvents()
    }

    class IDomainEvent {
        <<interface>>
    }

    class OrderCreatedDomainEvent {
        +OrderEntity Order
    }

    class OrderCancelledDomainEvent {
        +OrderEntity Order
        +string Reason
    }

    class OrderDeliveredDomainEvent {
        +OrderEntity Order
    }

    class CreateOrderCommand {
        +CreateOrUpdateOrderDto Dto
        +Actor Actor
    }

    class CreateOrderCommandHandler {
        -IUnitOfWork unitOfWork
        -ICatalogGrpcService catalog
        -IDiscountGrpcService discount
        +Handle(command, ct) Task~Guid~
    }

    class IUnitOfWork {
        <<interface>>
        +IOrderRepository Orders
        +SaveChangesAsync()
    }

    class ICatalogGrpcService {
        <<interface>>
        +GetProductByIdAsync(id) ProductDto
        +GetProductsAsync(ids, searchText) List~ProductDto~
        +GetAllAvailableProductsAsync(ids, searchText) List~ProductDto~
    }

    class IDiscountGrpcService {
        <<interface>>
        +ApplyCouponAsync(code, amount) ApplyCouponResponse
        +EvaluateCouponAsync(code, amount) EvaluateCouponResponse
    }

    %% Relationships
    Aggregate~T~ <|-- OrderEntity
    OrderEntity "1" --> "*" OrderItemEntity : contains
    OrderEntity --> Customer : has
    OrderEntity --> Address : has
    OrderEntity --> OrderNo : has
    OrderEntity --> Discount : has
    OrderEntity --> OrderStatus : has status
    OrderItemEntity --> Product : has
    
    IDomainEvent <|.. OrderCreatedDomainEvent
    IDomainEvent <|.. OrderCancelledDomainEvent
    IDomainEvent <|.. OrderDeliveredDomainEvent
    
    OrderEntity ..> OrderCreatedDomainEvent : raises
    OrderEntity ..> OrderCancelledDomainEvent : raises
    OrderEntity ..> OrderDeliveredDomainEvent : raises
    
    CreateOrderCommandHandler ..> CreateOrderCommand : handles
    CreateOrderCommandHandler --> IUnitOfWork : uses
    CreateOrderCommandHandler --> ICatalogGrpcService : uses
    CreateOrderCommandHandler --> IDiscountGrpcService : uses
    CreateOrderCommandHandler ..> OrderEntity : creates
```

---

## 7. State Machine - Order Status Flow

Sơ đồ trạng thái của đơn hàng từ khi tạo đến khi hoàn thành.

```mermaid
stateDiagram-v2
    [*] --> Pending: CreateOrderCommand
    
    Pending --> Processing: UpdateStatus
    Pending --> Cancelled: CancelOrder
    
    Processing --> Paid: Payment Complete
    Processing --> Cancelled: CancelOrder
    
    Paid --> Shipped: Ship Order
    Paid --> Cancelled: CancelOrder
    
    Shipped --> Delivered: Delivery Complete
    Shipped --> Cancelled: CancelOrder
    
    Delivered --> Refunded: RefundOrder
    
    Cancelled --> [*]
    Delivered --> [*]
    Refunded --> [*]

    Pending: Pending
    note right of Pending
        Đơn hàng mới tạo
        Chờ xử lý
    end note

    Processing: Processing
    note right of Processing
        Đang xử lý
        Kiểm tra tồn kho
    end note

    Paid: Paid
    note right of Paid
        Đã thanh toán
        Chờ giao hàng
    end note

    Shipped: Shipped
    note right of Shipped
        Đang giao hàng
        Vận chuyển
    end note

    Delivered: Delivered
    note right of Delivered
        Đã giao hàng
        Hoàn thành
    end note

    Cancelled: Cancelled
    note right of Cancelled
        Đã hủy
        Có lý do hủy
    end note

    Refunded: Refunded
    note right of Refunded
        Đã hoàn tiền
        Có lý do hoàn
    end note
```

---

## 8. Component Diagram - Order Service Internals

Sơ đồ thành phần chi tiết bên trong Order Service.

```mermaid
flowchart TB
    subgraph OrderApi["Order.Api (Presentation)"]
        Carter["Carter Modules"]
        Endpoints["Endpoints<br/>- CreateOrder<br/>- GetOrderById<br/>- UpdateOrder<br/>- GetMyOrders"]
        Auth["Authorization"]
    end

    subgraph OrderApp["Order.Application (Application)"]
        subgraph Commands["Commands"]
            CreateCmd["CreateOrderCommand"]
            UpdateCmd["UpdateOrderCommand"]
            UpdateStatusCmd["UpdateOrderStatusCommand"]
        end

        subgraph Queries["Queries"]
            GetById["GetOrderByIdQuery"]
            GetMyOrders["GetMyOrdersQuery"]
            GetAll["GetAllOrdersQuery"]
        end

        subgraph Handlers["Handlers"]
            CreateHandler["CreateOrderCommandHandler"]
            UpdateHandler["UpdateOrderCommandHandler"]
            GetHandler["GetOrderByIdQueryHandler"]
        end

        subgraph DomainHandlers["Domain Event Handlers"]
            OrderCreatedHandler["OrderCreatedDomainEventHandler"]
            OrderCancelledHandler["OrderCancelledDomainEventHandler"]
        end

        Dtos["DTOs & Mappings<br/>AutoMapper Profiles"]
        Validators["FluentValidation<br/>Validators"]
    end

    subgraph OrderDomain["Order.Domain (Domain)"]
        Aggregates["Aggregate Roots<br/>- OrderEntity"]
        Entities["Entities<br/>- OrderItemEntity"]
        VOs["Value Objects<br/>- Customer<br/>- Address<br/>- Product<br/>- Discount<br/>- OrderNo"]
        DomainEvents["Domain Events<br/>- OrderCreatedDomainEvent<br/>- OrderCancelledDomainEvent"]
        Repositories["Repository Interfaces<br/>- IOrderRepository"]
        Enums["Enums<br/>- OrderStatus"]
    end

    subgraph OrderInfra["Order.Infrastructure (Infrastructure)"]
        DbContext["ApplicationDbContext<br/>EF Core"]
        Repos["Repositories<br/>- OrderRepository<br/>- OrderItemRepository"]
        UoW["UnitOfWork"]
        GrpcClients["gRPC Clients<br/>- CatalogGrpcService<br/>- DiscountGrpcService"]
        Interceptors["Interceptors<br/>- DispatchDomainEvents<br/>- AuditableEntity"]
    end

    subgraph Workers["Background Workers"]
        OutboxWorker["Order.Worker.Outbox<br/>OutboxProcessor"]
        ConsumerWorker["Order.Worker.Consumer<br/>BasketCheckoutHandler"]
    end

    %% Api Layer
    Carter --> Endpoints
    Endpoints --> Auth
    Endpoints -->|"ISender.Send()"| CreateCmd
    Endpoints -->|"ISender.Send()"| GetById

    %% Application Layer
    CreateCmd -->|"Handled by"| CreateHandler
    UpdateCmd -->|"Handled by"| UpdateHandler
    GetById -->|"Handled by"| GetHandler
    
    CreateHandler -->|"Uses"| Dtos
    CreateHandler -->|"Validates"| Validators
    CreateHandler -->|"Creates"| Aggregates
    
    Aggregates -->|"Raises"| DomainEvents
    DomainEvents -->|"Handled by"| OrderCreatedHandler
    
    %% Domain Layer
    Aggregates -->|"Contains"| Entities
    Aggregates -->|"Has"| VOs
    Aggregates -->|"Uses"| Enums
    Repositories -->|"Manages"| Aggregates

    %% Infrastructure Layer
    CreateHandler -->|"Uses"| UoW
    UoW -->|"Uses"| Repos
    Repos -->|"Uses"| DbContext
    DbContext -->|"Has"| Interceptors
    
    CreateHandler -->|"Calls"| GrpcClients
    
    OrderCreatedHandler -->|"Saves to"| DbContext
    
    %% Workers
    DbContext -->|"Polls"| OutboxWorker
    OutboxWorker -->|"Publishes"| RabbitMQ
    RabbitMQ -->|"Consumes"| ConsumerWorker
    ConsumerWorker -->|"Sends"| CreateCmd

    %% External
    RabbitMQ[("RabbitMQ")]

    style OrderApi fill:#e3f2fd,stroke:#1565c0
    style OrderApp fill:#e8f5e9,stroke:#2e7d32
    style OrderDomain fill:#fff3e0,stroke:#e65100
    style OrderInfra fill:#f3e5f5,stroke:#4a148c
    style Workers fill:#fce4ec,stroke:#880e4f
```

---

## Tóm tắt kiến trúc

### Các thành phần chính trong workflow tạo đơn hàng:

| Thành phần | Mô tả | Vai trò |
|------------|-------|---------|
| **Order.Api** | Carter Endpoints | Tiếp nhận HTTP requests, xác thực |
| **CreateOrderCommand** | CQRS Command | Định nghĩa dữ liệu tạo order |
| **CreateOrderCommandHandler** | Command Handler | Xử lý business logic, gọi gRPC services |
| **OrderEntity** | Aggregate Root | Quản lý trạng thái và domain logic |
| **CatalogGrpcService** | gRPC Client | Lấy thông tin sản phẩm |
| **DiscountGrpcService** | gRPC Client | Áp dụng và tính toán coupon |
| **UnitOfWork** | Transaction Manager | Quản lý transaction và repositories |
| **OutboxProcessor** | Background Worker | Đảm bảo reliable message publishing |
| **BasketCheckoutIntegrationEvent** | Integration Event | Tạo order từ basket checkout |

### Design Patterns sử dụng:

1. **CQRS** - Phân tách Commands và Queries
2. **Domain-Driven Design (DDD)** - Aggregates, Value Objects, Domain Events
3. **Outbox Pattern** - Đảm bảo consistency giữa database và message broker
4. **Unit of Work** - Quản lý transaction
5. **gRPC** - Inter-service communication (Catalog, Discount)
6. **Event-Driven Architecture** - Domain Events và Integration Events
7. **Background Workers** - Xử lý async tasks (Outbox, Event Consumers)

### Luồng dữ liệu chính:

```
Client Request 
  → API Gateway 
  → Order.Api Endpoint 
  → MediatR 
  → CreateOrderCommandHandler 
  → gRPC (Catalog + Discount) 
  → OrderEntity (Domain Logic) 
  → UnitOfWork 
  → Database (Transactional Save)
  → Domain Event Raised
  → Outbox Message Saved
  → Outbox Processor (Background)
  → RabbitMQ
  → Event Consumers (Inventory, Communication)
```