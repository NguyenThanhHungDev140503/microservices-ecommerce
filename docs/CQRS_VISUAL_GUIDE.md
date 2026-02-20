# Phân Tích Cấu Trúc CQRS Trong Dự Án ProgCoder Shop Microservices

## Tài Liệu Trực Quan Hóa Bằng Flowchart

---

## Mục Lục

1. [Tổng Quan CQRS](#1-tổng-quan-cqrs)
2. [Kiến Trúc Tổng Thể](#2-kiến-trúc-tổng-thể)
3. [Command Side (Write Model)](#3-command-side-write-model)
4. [Query Side (Read Model)](#4-query-side-read-model)
5. [Pipeline Behaviors](#5-pipeline-behaviors)
6. [Domain Events và Đồng Bộ Dữ Liệu](#6-domain-events-và-đồng-bộ-dữ-liệu)
7. [Cấu Trúc Thư Mục](#7-cấu-trúc-thư-mục)
8. [Ví Dụ Thực Tế](#8-ví-dụ-thực-tế)

---

## 1. Tổng Quan CQRS

### 1.1. CQRS Là Gì?

```mermaid
flowchart TB
    subgraph "CQRS Pattern"
        direction TB
        
        subgraph "Traditional Architecture"
            T1[Single Model]
            T2[Handles Both Read & Write]
            T1 --> T2
        end
        
        subgraph "CQRS Architecture"
            direction LR
            C1[Command Side<br/>Write Model]
            C2[Query Side<br/>Read Model]
        end
        
        T2 -.->|"Tách biệt"| C1
        T2 -.->|"Tách biệt"| C2
    end
    
    style T1 fill:#ffcccc
    style T2 fill:#ffcccc
    style C1 fill:#ccffcc
    style C2 fill:#ccccff
```

### 1.2. Lý Do Áp Dụng

```mermaid
flowchart LR
    subgraph "Yêu Cầu Hệ Thống"
        A["Read/Write Ratio<br/>95 to 5"]
        B[Multiple<br/>Microservices]
        C[High Performance<br/>Reads]
        D[Event-Driven<br/>Architecture]
        E[Complex Business<br/>Logic]
    end
    
    subgraph "Giải Pháp CQRS"
        F[Scale Read/Write<br/>Independently]
        G[Optimize Queries<br/>Caching, Denormalization]
        H[Separate Business<br/>Rules]
        I[Loose Coupling<br/>via Events]
    end
    
    A --> F
    B --> I
    C --> G
    D --> I
    E --> H
```

---

## 2. Kiến Trúc Tổng Thể

### 2.1. Luồng Dữ Liệu CQRS

```mermaid
flowchart TB
    subgraph "Client Layer"
        CLIENT[Client<br/>Web/Mobile/App]
    end
    
    subgraph "API Gateway"
        GW[API Gateway<br/>YARP Reverse Proxy]
    end
    
    subgraph "Command Side - Write Model"
        direction TB
        CMD[Command<br/>DTO]
        VAL[Validation<br/>FluentValidation]
        CH[Command Handler<br/>Business Logic]
        DOM[Domain Layer<br/>Aggregates, Entities]
        REPO[Repository<br/>Write Operations]
        DB_WRITE[(Write Database<br/>PostgreSQL)]
        
        CMD --> VAL --> CH --> DOM --> REPO --> DB_WRITE
    end
    
    subgraph "Query Side - Read Model"
        direction TB
        QRY[Query<br/>DTO]
        QH[Query Handler<br/>Read Operations]
        REPO_Q[Repository<br/>Read Operations]
        CACHE[(Cache<br/>Redis)]
        DB_READ[(Read Database<br/>MongoDB/ES)]
        
        QRY --> QH --> REPO_Q
        REPO_Q --> CACHE
        REPO_Q --> DB_READ
    end
    
    subgraph "Event-Driven Integration"
        direction TB
        DE[Domain Events]
        OUTBOX[Outbox<br/>Pattern]
        MB[Message Broker<br/>RabbitMQ]
        CONSUMER[Event Consumers<br/>Other Services]
        READ_MODEL[Update Read Models]
        
        DE --> OUTBOX --> MB --> CONSUMER --> READ_MODEL
    end
    
    CLIENT --> GW
    GW --> CMD
    GW --> QRY
    DOM -.->|"Trigger"| DE
    READ_MODEL -.->|"Update"| DB_READ
    
    style CMD fill:#ffcccc
    style QRY fill:#ccccff
    style DE fill:#ffffcc
    style OUTBOX fill:#ffffcc
    style MB fill:#ffccff
```

### 2.2. Kiến Trúc Theo Services

```mermaid
flowchart LR
    subgraph "Microservices với CQRS"
        direction TB
        
        subgraph "Basket Service"
            B_CMD["Commands<br/>StoreBasket<br/>DeleteBasket<br/>Checkout"]
            B_QRY["Queries<br/>GetBasket"]
            B_DB[(MongoDB)]
        end
        
        subgraph "Catalog Service"
            C_CMD["Commands<br/>CreateProduct<br/>UpdateProduct<br/>DeleteProduct"]
            C_QRY["Queries<br/>GetProducts<br/>SearchProducts"]
            C_DB[(PostgreSQL<br/>+ Marten)]
        end
        
        subgraph "Order Service"
            O_CMD["Commands<br/>CreateOrder<br/>UpdateOrder<br/>CancelOrder"]
            O_QRY["Queries<br/>GetOrder<br/>GetMyOrders"]
            O_DB[(PostgreSQL)]
        end
        
        subgraph "Inventory Service"
            I_CMD["Commands<br/>CreateItem<br/>ReserveStock<br/>UpdateStock"]
            I_QRY["Queries<br/>GetInventory<br/>GetReservations"]
            I_DB[(PostgreSQL)]
        end
        
        subgraph "Report Service"
            R_CMD["Commands<br/>UpdateDashboard<br/>UpdateCharts"]
            R_QRY["Queries<br/>GetDashboard<br/>GetCharts"]
            R_DB[(MongoDB)]
        end
        
        subgraph "Search Service"
            S_CMD["Commands<br/>UpsertProduct<br/>DeleteProduct"]
            S_QRY["Queries<br/>SearchProduct"]
            S_DB[(Elasticsearch)]
        end
    end
    
    style B_CMD fill:#ffcccc
    style C_CMD fill:#ffcccc
    style O_CMD fill:#ffcccc
    style I_CMD fill:#ffcccc
    style R_CMD fill:#ffcccc
    style S_CMD fill:#ffcccc
    style B_QRY fill:#ccccff
    style C_QRY fill:#ccccff
    style O_QRY fill:#ccccff
    style I_QRY fill:#ccccff
    style R_QRY fill:#ccccff
    style S_QRY fill:#ccccff
```

---

## 3. Command Side (Write Model)

### 3.1. Cấu Trúc Command

```mermaid
flowchart TB
    subgraph "Command Structure"
        direction TB
        
        ICommand["ICommand-TResponse-<br/>Interface"]
        
        subgraph "Command Types"
            C1["CreateOrderCommand<br/>record - ICommand-Guid-"]
            C2["UpdateProductCommand<br/>record - ICommand-bool-"]
            C3["DeleteBasketCommand<br/>record - ICommand-Unit-"]
        end
        
        subgraph "Command Properties"
            P1["Dto - CreateOrderDto"]
            P2["Actor - User context"]
            P3["Immutable - record type"]
        end
        
        ICommand --> C1
        ICommand --> C2
        ICommand --> C3
        
        C1 --> P1
        C1 --> P2
        C1 --> P3
    end
    
    style ICommand fill:#ffffcc
    style C1 fill:#ffcccc
    style C2 fill:#ffcccc
    style C3 fill:#ffcccc
```

### 3.2. Command Handler Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as API Endpoint
    participant VAL as ValidationBehavior
    participant LOG as LoggingBehavior
    participant CH as Command Handler
    participant DOM as Domain Entity
    participant UOW as Unit of Work
    participant DB as Database
    participant DE as Domain Events
    participant OUT as Outbox
    
    Client->>API: Send Command
    API->>VAL: await mediator.Send(command)
    
    rect rgb(255, 200, 200)
        Note over VAL: Validation Pipeline
        VAL->>VAL: Validate with FluentValidation
        alt Validation Failed
            VAL--xClient: ValidationException
        end
    end
    
    VAL->>LOG: await next()
    
    rect rgb(200, 255, 200)
        Note over LOG: Logging Pipeline
        LOG->>LOG: Log [START] request
        LOG->>CH: await next()
    end
    
    rect rgb(200, 200, 255)
        Note over CH: Command Handler
        CH->>CH: 1. Validate input
        CH->>CH: 2. Call external services (gRPC)
        CH->>CH: 3. Create Domain Entity
        CH->>DOM: entity.Create()
        CH->>DOM: entity.AddItem()
        CH->>DOM: entity.TriggerEvent()
        DOM->>DE: AddDomainEvent()
        CH->>UOW: await unitOfWork.AddAsync(entity)
        CH->>UOW: await unitOfWork.SaveChangesAsync()
        UOW->>DB: INSERT/UPDATE
    end
    
    CH-->>LOG: Return result
    LOG->>LOG: Log [END] request
    LOG-->>VAL: Return result
    VAL-->>API: Return result
    API-->>Client: Return response
    
    Note over DE,OUT: After SaveChanges
    DE->>OUT: Domain Event Handler<br/>creates Outbox message
```

### 3.3. Command Handler Chi Tiết

```mermaid
flowchart TB
    subgraph "CreateOrderCommandHandler Flow"
        direction TB
        
        START([Start])
        
        subgraph "Input Validation"
            V1["Validate Dto.Customer"]
            V2["Validate Dto.ShippingAddress"]
            V3["Validate Dto.OrderItems"]
        end
        
        subgraph "External Service Calls"
            E1["catalogGrpc.GetAllAvailableProducts()"]
            E2["discountGrpc.EvaluateCoupon()"]
            E3["discountGrpc.ApplyCoupon()"]
        end
        
        subgraph "Domain Logic"
            D1["Customer.Of()<br/>Create Value Object"]
            D2["Address.Of()<br/>Create Value Object"]
            D3["OrderEntity.Create()<br/>Create Aggregate"]
            D4["order.AddOrderItem()<br/>Add Items"]
            D5["order.ApplyDiscount()<br/>Apply Discount"]
            D6["order.OrderCreated()<br/>Trigger Domain Event"]
        end
        
        subgraph "Persistence"
            P1["unitOfWork.Orders.AddAsync()"]
            P2["unitOfWork.SaveChangesAsync()"]
        end
        
        END([Return OrderId])
        
        START --> V1 --> V2 --> V3
        V3 --> E1
        E1 --> E2
        E2 -->|if has coupon| E3
        E3 --> D1
        E1 -->|if no coupon| D1
        D1 --> D2 --> D3 --> D4 --> D5 --> D6
        D6 --> P1 --> P2 --> END
    end
    
    style START fill:#ccffcc
    style END fill:#ccffcc
    style V1 fill:#ffcccc
    style V2 fill:#ffcccc
    style V3 fill:#ffcccc
    style E1 fill:#ccccff
    style E2 fill:#ccccff
    style E3 fill:#ccccff
    style D1 fill:#ffffcc
    style D2 fill:#ffffcc
    style D3 fill:#ffffcc
    style D4 fill:#ffffcc
    style D5 fill:#ffffcc
    style D6 fill:#ffffcc
    style P1 fill:#ccffff
    style P2 fill:#ccffff
```

---

## 4. Query Side (Read Model)

### 4.1. Cấu Trúc Query

```mermaid
flowchart TB
    subgraph "Query Structure"
        direction TB
        
        IQuery["IQuery-TResponse-<br/>Interface"]
        
        subgraph "Query Types"
            Q1["GetOrderByIdQuery<br/>record - IQuery-GetOrderByIdResult-"]
            Q2["GetMyOrdersQuery<br/>record - IQuery-GetMyOrdersResult-"]
            Q3["SearchProductQuery<br/>record - IQuery-SearchProductResult-"]
        end
        
        subgraph "Query Properties"
            P1["OrderId - Guid"]
            P2["Filter - OrderFilter"]
            P3["Paging - PaginationRequest"]
        end
        
        subgraph "Result Types"
            R1["GetOrderByIdResult<br/>record with OrderDto"]
            R2["GetMyOrdersResult<br/>record with List-OrderDto-"]
        end
        
        IQuery --> Q1
        IQuery --> Q2
        IQuery --> Q3
        
        Q1 --> P1
        Q2 --> P2
        Q2 --> P3
        
        Q1 --> R1
        Q2 --> R2
    end
    
    style IQuery fill:#ffffcc
    style Q1 fill:#ccccff
    style Q2 fill:#ccccff
    style Q3 fill:#ccccff
    style R1 fill:#ccffcc
    style R2 fill:#ccffcc
```

### 4.2. Query Handler Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as API Endpoint
    participant LOG as LoggingBehavior
    participant QH as Query Handler
    participant REPO as Repository
    participant CACHE as Cache
    participant DB as Database
    participant MAP as AutoMapper
    
    Client->>API: Send Query
    API->>LOG: await mediator.Send(query)
    
    rect rgb(200, 255, 200)
        Note over LOG: Logging Pipeline
        LOG->>LOG: Log [START] request
        LOG->>QH: await next()
    end
    
    rect rgb(200, 200, 255)
        Note over QH: Query Handler
        
        alt Simple Query
            QH->>REPO: await repo.GetByIdAsync()
            REPO->>CACHE: Check cache
            CACHE-->>REPO: Cache miss
            REPO->>DB: SELECT with AsNoTracking()
            DB-->>REPO: Entity
            REPO-->>QH: Entity
            QH->>MAP: mapper.Map-Dto-(entity)
            MAP-->>QH: Dto
        end
        
        alt Complex Query with Filters
            QH->>QH: Build IQueryable
            QH->>QH: Apply WhereIf filters
            QH->>QH: Apply OrderBy
            QH->>QH: Count total items
            QH->>QH: Apply WithPaging
            QH->>REPO: await query.ToListAsync()
            REPO->>DB: SELECT (optimized)
            DB-->>REPO: List-Entity-
            REPO-->>QH: List-Entity-
            QH->>MAP: mapper.Map-List-Dto-(entities)
            MAP-->>QH: List-Dto-
        end
        
        alt Query with External Data
            QH->>REPO: Get basket from DB
            REPO-->>QH: BasketEntity
            QH->>GRPC: catalogGrpc.GetProductsByIds()
            GRPC-->>QH: Product details
            QH->>QH: Enrich basket items
        end
    end
    
    QH-->>LOG: Return result
    LOG->>LOG: Log [END] request
    LOG-->>API: Return result
    API-->>Client: Return response
```

### 4.3. Query Optimization Strategies

```mermaid
flowchart TB
    subgraph "Query Optimization"
        direction TB
        
        subgraph "1. AsNoTracking"
            A1["Read-Only Queries"]
            A2["No Change Tracking"]
            A3["Better Performance"]
            A1 --> A2 --> A3
        end
        
        subgraph "2. Projection"
            B1["Select Only Needed Fields"]
            B2["Reduce Data Transfer"]
            B3["Faster Queries"]
            B1 --> B2 --> B3
        end
        
        subgraph "3. Caching"
            C1["CachedBasketRepository"]
            C2["MemoryCache / Redis"]
            C3["Cache-Aside Pattern"]
            C1 --> C2 --> C3
        end
        
        subgraph "4. Pagination"
            D1["WithPaging Extension"]
            D2["Skip + Take"]
            D3["Avoid Large Result Sets"]
            D1 --> D2 --> D3
        end
        
        subgraph "5. Denormalization"
            E1["Pre-aggregated Data"]
            E2["Read Models in MongoDB"]
            E3["No JOINs Needed"]
            E1 --> E2 --> E3
        end
    end
    
    style A1 fill:#ccffcc
    style B1 fill:#ccffcc
    style C1 fill:#ccffcc
    style D1 fill:#ccffcc
    style E1 fill:#ccffcc
```

---

## 5. Pipeline Behaviors

### 5.1. MediatR Pipeline

```mermaid
flowchart TB
    subgraph "MediatR Request Pipeline"
        direction TB
        
        IN([Incoming Request])
        
        subgraph "Pipeline Behaviors"
            direction TB
            B1["1.ValidationBehavior FluentValidation"]
            B2["2.LoggingBehavior Performance Logging"]
            B3["3.Handler Execution Business Logic"]
        end
        
        OUT([Response])
        
        IN --> B1
        
        B1 -->|Valid| B2
        B1 -->|Invalid| ERR[Throw ValidationException]
        
        B2 --> B3
        B3 --> OUT
        
        ERR -.->|Return| OUT
    end
    
    style IN fill:#ccffcc
    style OUT fill:#ccffcc
    style B1 fill:#ffcccc
    style B2 fill:#ccccff
    style B3 fill:#ffffcc
    style ERR fill:#ff9999
```

### 5.2. Validation Behavior Chi Tiết

```mermaid
flowchart TB
    subgraph "ValidationBehavior Flow"
        direction TB
        
        START([Request Received])
        
        subgraph "Validation Process"
            V1["Create ValidationContext"]
            V2["Run All Validators"]
            V3["Collect Failures"]
            V4["Any Failures?"]
        end
        
        V5["Throw ValidationException"]
        V6["await next()"]
        
        END_HANDLER([Handler Executes])
        END_ERROR([Error Response])
        
        START --> V1 --> V2 --> V3 --> V4
        
        V4 -->|Yes| V5 --> END_ERROR
        V4 -->|No| V6 --> END_HANDLER
    end
    
    style START fill:#ccffcc
    style END_HANDLER fill:#ccffcc
    style END_ERROR fill:#ff9999
    style V4 fill:#ffffcc
```

### 5.3. Logging Behavior Chi Tiết

```mermaid
flowchart TB
    subgraph "LoggingBehavior Flow"
        direction TB
        
        START([Request Received])
        
        L1["Log [START]<br/>RequestType, ResponseType, RequestData"]
        
        subgraph "Timer"
            T1["Start Stopwatch"]
            T2["Execute Handler"]
            T3["Stop Stopwatch"]
        end
        
        L2{"TimeTaken > 3s?"}
        L3["Log [PERFORMANCE] Warning"]
        L4["Log [END]<br/>RequestType, ResponseType"]
        
        END([Return Response])
        
        START --> L1 --> T1 --> T2
        T2 --> T3 --> L2
        L2 -->|Yes| L3 --> L4
        L2 -->|No| L4
        L4 --> END
    end
    
    style START fill:#ccffcc
    style END fill:#ccffcc
    style L2 fill:#ffffcc
    style L3 fill:#ffcc99
```

---

## 6. Domain Events và Đồng Bộ Dữ Liệu

### 6.1. Domain Events Flow

```mermaid
sequenceDiagram
    participant CH as Command Handler
    participant AGG as Aggregate Root
    participant UOW as UnitOfWork
    participant INT as DispatchDomainEventsInterceptor
    participant MED as MediatR
    participant DEH as Domain Event Handler
    participant OUT as Outbox
    participant BG as Background Worker
    participant MB as Message Broker
    participant CON as Consumer Service
    
    CH->>AGG: entity.OrderCreated()
    AGG->>AGG: AddDomainEvent(new OrderCreatedDomainEvent())
    
    CH->>UOW: await unitOfWork.SaveChangesAsync()
    
    rect rgb(255, 255, 200)
        Note over INT: EF Core Interceptor
        UOW->>INT: Intercepts SaveChanges
        INT->>INT: Get aggregates with domain events
        INT->>INT: Extract domain events
        INT->>INT: Clear events from aggregates
        INT->>MED: await mediator.Publish(events)
    end
    
    MED->>DEH: Handle OrderCreatedDomainEvent
    
    rect rgb(255, 200, 200)
        Note over DEH: Domain Event Handler
        DEH->>DEH: Create Integration Event
        DEH->>OUT: Create Outbox Message
    end
    
    rect rgb(200, 255, 200)
        Note over BG: Background Processing
        BG->>OUT: Poll pending messages
        OUT->>BG: Outbox messages
        BG->>MB: Publish to RabbitMQ
    end
    
    MB->>CON: Consume Integration Event
    CON->>CON: Update Read Model
```

### 6.2. Outbox Pattern

```mermaid
flowchart TB
    subgraph "Outbox Pattern - Reliable Event Delivery"
        direction TB
        
        subgraph "Write Side (Transaction)"
            T1["Begin Transaction"]
            T2["Save Business Data"]
            T3["Save Outbox Message"]
            T4["Commit Transaction"]
            
            T1 --> T2 --> T3 --> T4
        end
        
        subgraph "Outbox Table"
            O1["Id - Guid"]
            O2["EventType - string"]
            O3["Content - JSON"]
            O4["OccurredOnUtc - DateTime"]
            O5["ProcessedOnUtc - DateTime?"]
            O6["AttemptCount - int"]
            O7["LastErrorMessage - string?"]
        end
        
        subgraph "Background Worker"
            B1["Poll Pending Messages"]
            B2{"AttemptCount < 5?"}
            B3["Publish to Message Broker"]
            B4["Mark as Processed"]
            B5["Schedule Retry with Backoff"]
            B6["Mark as Failed"]
            
            B1 --> B2
            B2 -->|Yes| B3 --> B4
            B2 -->|No| B6
            B3 -->|Failed| B5
        end
        
        T4 -.->|"Same Transaction"| O1
        O1 -.->|"Poll"| B1
    end
    
    style T1 fill:#ccffcc
    style T4 fill:#ccffcc
    style B4 fill:#ccffcc
    style B6 fill:#ff9999
    style B2 fill:#ffffcc
```

### 6.3. Inbox Pattern (Idempotency)

```mermaid
flowchart TB
    subgraph "Inbox Pattern - Idempotent Event Processing"
        direction TB
        
        START([Integration Event Received])
        
        subgraph "Idempotency Check"
            I1["Extract MessageId"]
            I2["Check Inbox Table"]
            I3{"Message Exists?"}
        end
        
        I4["Log - Already Processed"]
        I5["Create Inbox Record"]
        
        subgraph "Process Event"
            P1["Update Inventory"]
            P2["Send Notification"]
            P3["Update Report Data"]
        end
        
        END([Done])
        
        START --> I1 --> I2 --> I3
        I3 -->|Yes| I4 --> END
        I3 -->|No| I5 --> P1 --> P2 --> P3 --> END
    end
    
    style START fill:#ccffcc
    style END fill:#ccffcc
    style I3 fill:#ffffcc
    style I4 fill:#ffcc99
    style I5 fill:#ccffcc
```

### 6.4. DispatchDomainEventsInterceptor Chi Tiết

```mermaid
flowchart TB
    subgraph "EF Core Interceptor Flow"
        direction TB
        
        START([SaveChanges Called])
        
        subgraph "Interceptor Steps"
            I1["1.Get aggregates with domain events"]
            I2["2.Extract domain events"]
            I3["3.Clear events from aggregates"]
            I4["4.Publish via MediatR"]
        end
        
        subgraph "ChangeTracker Query"
            CT1["context.ChangeTracker"]
            CT2["Entries-IAggregate-"]
            CT3["Where events Any"]
            CT4["Select Entity"]
        end
        
        subgraph "Event Processing"
            EP1["SelectMany DomainEvents"]
            EP2["ToList"]
            EP3["ForEach ClearDomainEvents"]
            EP4["foreach Publish"]
        end
        
        END([Continue SaveChanges])
        
        START --> I1
        I1 --> CT1 --> CT2 --> CT3 --> CT4
        CT4 --> I2 --> EP1 --> EP2
        EP2 --> I3 --> EP3 --> I4 --> EP4 --> END
    end
    
    style START fill:#ccffcc
    style END fill:#ccffcc
    style I1 fill:#ffffcc
    style I2 fill:#ffffcc
    style I3 fill:#ffffcc
    style I4 fill:#ffffcc
```

### 6.5. Write vs Read Model - Normalized vs Denormalized

```mermaid
flowchart TB
    subgraph "CQRS Data Synchronization"
        direction LR
        
        subgraph "Write Side - Normalized"
            direction TB
            W1["Orders Table"]
            W2["OrderItems Table"]
            W3["Customers Table"]
            W4["Products Table"]
            
            W1 --> W2
            W1 --> W3
            W2 --> W4
        end
        
        subgraph "Sync Mechanism"
            direction TB
            E1["Domain Events"]
            E2["Outbox Pattern"]
            E3["RabbitMQ"]
            E4["Event Consumers"]
            
            E1 --> E2 --> E3 --> E4
        end
        
        subgraph "Read Side - Denormalized"
            direction TB
            R1["DashboardTotals"]
            R2["OrderSummaries"]
            R3["ProductSearch"]
            R4["ReportData"]
            
            R1
            R2
            R3
            R4
        end
        
        W1 -.->|"Triggers"| E1
        E4 -.->|"Updates"| R1
        E4 -.->|"Updates"| R2
        E4 -.->|"Updates"| R3
        E4 -.->|"Updates"| R4
    end
    
    style W1 fill:#ffcccc
    style W2 fill:#ffcccc
    style W3 fill:#ffcccc
    style W4 fill:#ffcccc
    style R1 fill:#ccccff
    style R2 fill:#ccccff
    style R3 fill:#ccccff
    style R4 fill:#ccccff
    style E1 fill:#ffffcc
    style E2 fill:#ffffcc
    style E3 fill:#ffccff
```

### 6.6. So Sánh Normalized vs Denormalized

```mermaid
flowchart TB
    subgraph "Normalized vs Denormalized Comparison"
        direction LR
        
        subgraph "Normalized - Write Model"
            direction TB
            N1["Single Source of Truth"]
            N2["ACID Transactions"]
            N3["Complex Queries"]
            N4["JOINs Required"]
            N5["PostgreSQL"]
            
            N1 --> N2 --> N3 --> N4 --> N5
        end
        
        subgraph "Denormalized - Read Model"
            direction TB
            D1["Pre-aggregated Data"]
            D2["Eventual Consistency"]
            D3["Simple Queries"]
            D4["No JOINs"]
            D5["MongoDB / ES"]
            
            D1 --> D2 --> D3 --> D4 --> D5
        end
        
        subgraph "Trade-offs"
            T1["Consistency"]
            T2["Performance"]
            T3["Complexity"]
            T4["Storage"]
        end
        
        N1 -.->|"Strong"| T1
        D1 -.->|"Eventual"| T1
        N3 -.->|"Slower"| T2
        D3 -.->|"Faster"| T2
        N4 -.->|"High"| T3
        D4 -.->|"Low"| T3
        N5 -.->|"Less"| T4
        D5 -.->|"More"| T4
    end
    
    style N1 fill:#ffcccc
    style D1 fill:#ccccff
    style T1 fill:#ffffcc
    style T2 fill:#ffffcc
    style T3 fill:#ffffcc
    style T4 fill:#ffffcc
```

### 6.7. Hybrid Approach - Kiến Trúc Thực Tế

```mermaid
flowchart TB
    subgraph "Hybrid CQRS Architecture"
        direction TB
        
        subgraph "Write Side"
            direction LR
            W1["Relational Database"]
            W2["Domain Events"]
            W3["Outbox Pattern"]
            
            W1 --> W2 --> W3
        end
        
        subgraph "Read Side Options"
            direction LR
            R1["Direct DB Queries"]
            R2["Cached Results"]
            R3["Pre-aggregated Views"]
            R4["Specialized Stores"]
        end
        
        subgraph "Use Cases"
            direction LR
            U1["Order Details"]
            U2["Basket Service"]
            U3["Dashboard"]
            U4["Product Search"]
        end
        
        W3 -.->|"Events"| R1
        W3 -.->|"Events"| R2
        W3 -.->|"Events"| R3
        W3 -.->|"Events"| R4
        
        R1 -.->|"Simple Read"| U1
        R2 -.->|"Frequent Access"| U2
        R3 -.->|"Analytics"| U3
        R4 -.->|"Full-text"| U4
    end
    
    style W1 fill:#ffcccc
    style W2 fill:#ffffcc
    style W3 fill:#ccffcc
    style R1 fill:#ccccff
    style R2 fill:#ccccff
    style R3 fill:#ccccff
    style R4 fill:#ccccff
```

---

## 7. Cấu Trúc Thư Mục

### 7.1. Project Structure

```mermaid
flowchart TB
    subgraph "CQRS Project Structure"
        direction TB
        
        ROOT["src/Services/{ServiceName}/"]
        
        subgraph "Application Layer"
            APP["{Service}.Application/"]
            
            subgraph "Features"
                F1["Order/"]
                F2["Product/"]
                F3["Basket/"]
            end
            
            subgraph "Commands"
                C1["CreateOrderCommand.cs"]
                C2["UpdateOrderCommand.cs"]
                C3["DeleteOrderCommand.cs"]
            end
            
            subgraph "Queries"
                Q1["GetOrderByIdQuery.cs"]
                Q2["GetMyOrdersQuery.cs"]
                Q3["GetAllOrdersQuery.cs"]
            end
            
            DTO["Dtos/"]
            MAP["Mappings/"]
        end
        
        subgraph "Domain Layer"
            DOM["{Service}.Domain/"]
            ENT["Entities/"]
            VO["ValueObjects/"]
            EV["Events/"]
            EX["Exceptions/"]
        end
        
        subgraph "Infrastructure Layer"
            INF["{Service}.Infrastructure/"]
            DATA["Data/"]
            REPO["Repositories/"]
            GRPC["gRPC Services/"]
        end
        
        ROOT --> APP
        ROOT --> DOM
        ROOT --> INF
        
        APP --> F1
        F1 --> C1
        F1 --> Q1
        APP --> DTO
        APP --> MAP
        
        DOM --> ENT
        DOM --> VO
        DOM --> EV
        DOM --> EX
        
        INF --> DATA
        INF --> REPO
        INF --> GRPC
    end
    
    style ROOT fill:#ffffcc
    style APP fill:#ccccff
    style DOM fill:#ccffcc
    style INF fill:#ffcccc
```

### 7.2. BuildingBlocks Structure

```mermaid
flowchart TB
    subgraph "Shared BuildingBlocks"
        direction TB
        
        BB["BuildingBlocks/"]
        
        subgraph "CQRS"
            C1["ICommand.cs"]
            C2["ICommandHandler.cs"]
            C3["IQuery.cs"]
            C4["IQueryHandler.cs"]
        end
        
        subgraph "Behaviors"
            B1["ValidationBehavior.cs"]
            B2["LoggingBehavior.cs"]
        end
        
        subgraph "Domain"
            D1["Entity.cs"]
            D2["Aggregate.cs"]
            D3["IDomainEvent.cs"]
        end
        
        subgraph "Pagination"
            P1["PaginatedResult.cs"]
            P2["PagingExtensions.cs"]
        end
        
        BB --> C1
        BB --> C2
        BB --> C3
        BB --> C4
        BB --> B1
        BB --> B2
        BB --> D1
        BB --> D2
        BB --> D3
        BB --> P1
        BB --> P2
    end
    
    style BB fill:#ffffcc
    style C1 fill:#ccccff
    style C2 fill:#ccccff
    style C3 fill:#ccccff
    style C4 fill:#ccccff
    style B1 fill:#ccffcc
    style B2 fill:#ccffcc
```

---

## 8. Ví Dụ Thực Tế

### 8.1. CreateOrder Flow - End to End

```mermaid
flowchart TB
    subgraph "Create Order - Complete Flow"
        direction TB
        
        CLIENT["Client<br/>POST /api/orders"]
        
        subgraph "Order Service"
            API["Order.Api<br/>Carter Module"]
            
            subgraph "Application Layer"
                CMD["CreateOrderCommand<br/>record"]
                VAL["CreateOrderCommandValidator<br/>FluentValidation"]
                HANDLER["CreateOrderCommandHandler"]
            end
            
            subgraph "Domain Layer"
                ENTITY["OrderEntity<br/>Aggregate Root"]
                EVENT["OrderCreatedDomainEvent"]
            end
            
            subgraph "Infrastructure Layer"
                UOW["UnitOfWork"]
                REPO["OrderRepository"]
                DB[(PostgreSQL)]
                OUTBOX["OutboxMessage"]
            end
        end
        
        subgraph "External Services"
            CATALOG["Catalog Service<br/>gRPC"]
            DISCOUNT["Discount Service<br/>gRPC"]
        end
        
        subgraph "Event Processing"
            BG["Background Worker"]
            RMQ["RabbitMQ"]
            INV["Inventory Service<br/>Consumer"]
            NOTIF["Notification Service<br/>Consumer"]
            REPORT["Report Service<br/>Consumer"]
        end
        
        CLIENT --> API
        API --> CMD
        CMD --> VAL
        VAL -->|Valid| HANDLER
        
        HANDLER --> CATALOG
        HANDLER --> DISCOUNT
        
        HANDLER --> ENTITY
        ENTITY --> EVENT
        
        HANDLER --> UOW
        UOW --> REPO
        REPO --> DB
        
        EVENT --> OUTBOX
        OUTBOX -.->|Poll| BG
        BG --> RMQ
        RMQ --> INV
        RMQ --> NOTIF
        RMQ --> REPORT
    end
    
    style CLIENT fill:#ccffcc
    style API fill:#ccccff
    style HANDLER fill:#ffffcc
    style ENTITY fill:#ffcccc
    style DB fill:#ccffff
    style RMQ fill:#ffccff
    style INV fill:#ccffcc
    style NOTIF fill:#ccffcc
    style REPORT fill:#ccffcc
```

### 8.2. GetOrder Flow - End to End

```mermaid
flowchart TB
    subgraph "Get Order - Complete Flow"
        direction TB
        
        CLIENT["Client<br/>GET /api/orders/{id}"]
        
        subgraph "Order Service"
            API["Order.Api<br/>Carter Module"]
            
            subgraph "Application Layer"
                QRY["GetOrderByIdQuery<br/>record"]
                HANDLER["GetOrderByIdQueryHandler"]
            end
            
            subgraph "Infrastructure Layer"
                UOW["UnitOfWork"]
                REPO["OrderRepository"]
                DB[(PostgreSQL)]
                MAP["AutoMapper"]
            end
        end
        
        subgraph "Response"
            DTO["OrderDto"]
            RESULT["GetOrderByIdResult"]
        end
        
        CLIENT --> API
        API --> QRY
        QRY --> HANDLER
        
        HANDLER --> UOW
        UOW --> REPO
        REPO -->|GetByIdWithRelationshipAsync| DB
        DB -->|OrderEntity| REPO
        REPO -->|Entity| HANDLER
        
        HANDLER -->|Map| MAP
        MAP -->|OrderDto| DTO
        DTO --> RESULT
        RESULT --> API
        API --> CLIENT
    end
    
    style CLIENT fill:#ccffcc
    style API fill:#ccccff
    style HANDLER fill:#ffffcc
    style DB fill:#ccffff
    style DTO fill:#ccffcc
```

### 8.3. Data Flow Comparison

```mermaid
flowchart TB
    subgraph "Command vs Query Data Flow"
        direction LR
        
        subgraph "Command Flow (Write)"
            direction TB
            C1["API Endpoint"]
            C2["Command"]
            C3["Validation"]
            C4["Handler"]
            C5["Domain Logic"]
            C6["SaveChanges"]
            C7["Domain Events"]
            C8["Outbox"]
            C9["Message Broker"]
            
            C1 --> C2 --> C3 --> C4 --> C5 --> C6 --> C7 --> C8 --> C9
        end
        
        subgraph "Query Flow (Read)"
            direction TB
            Q1["API Endpoint"]
            Q2["Query"]
            Q3["Handler"]
            Q4["Repository"]
            Q5["Cache Check"]
            Q6["Database"]
            Q7["Mapping"]
            Q8["Result"]
            
            Q1 --> Q2 --> Q3 --> Q4
            Q4 --> Q5
            Q5 -->|Cache Hit| Q7
            Q5 -->|Cache Miss| Q6 --> Q7
            Q7 --> Q8
        end
        
        C1 -.->|"Different Paths"| Q1
    end
    
    style C1 fill:#ffcccc
    style C2 fill:#ffcccc
    style C3 fill:#ffcccc
    style C4 fill:#ffcccc
    style C5 fill:#ffcccc
    style C6 fill:#ffcccc
    style C7 fill:#ffcccc
    style C8 fill:#ffcccc
    style C9 fill:#ffcccc
    
    style Q1 fill:#ccccff
    style Q2 fill:#ccccff
    style Q3 fill:#ccccff
    style Q4 fill:#ccccff
    style Q5 fill:#ccccff
    style Q6 fill:#ccccff
    style Q7 fill:#ccccff
    style Q8 fill:#ccccff
```

---

## Tóm Tắt

### Kiến Trúc CQRS Trong Dự Án

```mermaid
flowchart TB
    subgraph "CQRS Architecture Summary"
        direction TB
        
        A["Clean Architecture<br/>+ CQRS + DDD"]
        
        B["Command Side"]
        C["Query Side"]
        
        D["MediatR Pipeline"]
        E["Domain Events"]
        
        F["Outbox Pattern"]
        G["Inbox Pattern"]
        
        H["Event-Driven<br/>Microservices"]
        
        A --> B
        A --> C
        
        B --> D
        C --> D
        
        B --> E
        E --> F
        F --> H
        
        H --> G
    end
    
    style A fill:#ffffcc
    style B fill:#ffcccc
    style C fill:#ccccff
    style D fill:#ccffcc
    style E fill:#ffffcc
    style F fill:#ffccff
    style G fill:#ffccff
    style H fill:#ccffff
```

### Các Thành Phần Chính

| Thành Phần | Mục Đích | Vị Trí |
|------------|----------|--------|
| **ICommand/IQuery** | Định nghĩa contracts | BuildingBlocks.CQRS |
| **Command/Query** | DTOs chứa request data | Features/{Feature}/ |
| **Handlers** | Business logic | Features/{Feature}/ |
| **Validators** | Input validation | Cùng file với Command |
| **Behaviors** | Cross-cutting concerns | BuildingBlocks.Behaviors |
| **Domain Events** | Loose coupling | Domain/Events/ |
| **Outbox** | Reliable messaging | Infrastructure/Data/ |

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** AI Assistant - Visual CQRS Documentation for ProgCoder Shop Microservices
