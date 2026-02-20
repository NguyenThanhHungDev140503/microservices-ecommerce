# Tài Liệu Hệ Thống Unit of Work (UoW)

## A. Tổng Quan Hệ Thống

### 1. Giới Thiệu

Hệ thống **Unit of Work (UoW)** trong dự án ProgCoder Shop Microservices là một pattern quan trọng được triển khai theo nguyên tắc **Clean Architecture** và **Domain-Driven Design (DDD)**. Pattern này đảm bảo tính nhất quán của dữ liệu khi thực hiện nhiều thao tác trên nhiều repository khác nhau trong một transaction duy nhất.

### 2. Mục Tiêu Thiết Kế

- **Quản lý transaction**: Tập trung điều khiển transaction tại một điểm duy nhất
- **Đảm bảo tính nhất quán**: Tất cả thay đổi được commit hoặc rollback đồng thởi
- **Tách biệt concerns**: Tách biệt logic nghiệp vụ khỏi infrastructure concerns
- **Hỗ trợ đa database**: Hoạt động với cả MongoDB và Entity Framework Core
- **Hỗ trợ Outbox/Inbox pattern**: Tích hợp với hệ thống messaging

### 3. Kiến Trúc Tổng Thể

```mermaid
flowchart TB
    subgraph Application["Application Layer"]
        CH[Command Handlers]
        QH[Query Handlers]
        EH[Event Handlers]
    end
    
    subgraph Interface["Interface Layer"]
        IU[IUnitOfWork Interface]
    end
    
    subgraph Infrastructure["Infrastructure Layer"]
        UOW[UnitOfWork Implementation]
        subgraph Repos["Repository Layer"]
            R1[EF Core Repositories]
            R2[MongoDB Repositories]
        end
    end
    
    subgraph Data["Data Layer"]
        DB1[(SQL Server/PostgreSQL)]
        DB2[(MongoDB)]
    end
    
    CH --> IU
    QH --> IU
    EH --> IU
    IU --> UOW
    UOW --> R1
    UOW --> R2
    R1 --> DB1
    R2 --> DB2
```

## B. Chi Tiết Triển Khai

### 1. Cấu Trúc Interface IUnitOfWork

#### 1.1 Discount Service (MongoDB)

```mermaid
classDiagram
    class IUnitOfWork {
        <<interface>>
        +ICouponRepository Coupons
        +BeginTransactionAsync() Task
        +CommitTransactionAsync() Task
        +RollbackTransactionAsync() Task
        +SaveChangesAsync() Task~int~
    }
    
    class ICouponRepository {
        <<interface>>
    }
    
    class IDisposable {
        <<interface>>
    }
    
    class IAsyncDisposable {
        <<interface>>
    }
    
    IUnitOfWork --> ICouponRepository : exposes
    IUnitOfWork --|> IDisposable : implements
    IUnitOfWork --|> IAsyncDisposable : implements
```

**File**: `Discount.Application/Repositories/IUnitOfWork.cs`

**Đặc điểm:**
- Kế thừa `IDisposable` và `IAsyncDisposable` để quản lý tài nguyên
- Hỗ trợ MongoDB transactions với explicit methods
- Property `Coupons` cung cấp truy cập đến repository

#### 1.2 Inventory Service (EF Core)

```mermaid
classDiagram
    class IUnitOfWork {
        <<interface>>
        +IInventoryReservationRepository InventoryReservations
        +IInventoryItemRepository InventoryItems
        +IInventoryHistoryRepository InventoryHistories
        +ILocationRepository Locations
        +IInboxMessageRepository InboxMessages
        +IOutboxMessageRepository OutboxMessages
        +SaveChangesAsync() Task~int~
        +BeginTransactionAsync() Task~IDbTransaction~
    }
    
    IUnitOfWork --> "6" IRepository : aggregates
```

**File**: `Inventory.Domain/Abstractions/IUnitOfWork.cs`

**Đặc điểm:**
- Sử dụng EF Core với `IDbTransaction`
- Tích hợp Outbox/Inbox pattern với `InboxMessages` và `OutboxMessages`
- 6 repositories cho các domain entities khác nhau

#### 1.3 Order Service (EF Core)

Cấu trúc tương tự Inventory Service với 4 repositories:
- `IOrderRepository Orders`
- `IOrderItemRepository OrderItems`
- `IInboxMessageRepository InboxMessages`
- `IOutboxMessageRepository OutboxMessages`

### 2. Implementations

#### 2.1 MongoDB UnitOfWork (Discount Service)

```mermaid
sequenceDiagram
    participant Client
    participant UoW as UnitOfWork
    participant Session as MongoSession
    participant Repo as CouponRepository
    participant DB as MongoDB

    Client->>UoW: BeginTransactionAsync()
    UoW->>Session: StartSessionAsync()
    Session->>Session: StartTransaction()
    Session-->>UoW: IClientSessionHandle
    
    Client->>Repo: AddAsync(coupon)
    Repo->>Session: GetSession()
    Repo->>DB: Insert with session
    
    Client->>UoW: SaveChangesAsync()
    UoW->>Session: CommitTransactionAsync()
    Session->>DB: Commit
    UoW->>Session: Dispose()
    
    alt Exception
        Client->>UoW: RollbackTransactionAsync()
        UoW->>Session: AbortTransactionAsync()
        UoW->>Session: Dispose()
    end
```

**File**: `Discount.Infrastructure/Repositories/UnitOfWork.cs`

**Giải thích chi tiết:**

1. **IMongoSessionProvider**: Interface cho phép repository truy cập session hiện tại
2. **Transaction Management**: 
   - Session được tạo và gắn với transaction
   - Repositories sử dụng session này cho mọi operations
   - Commit/Rollback thông qua session
3. **SaveChangesAsync**: Tự động commit nếu có transaction active
4. **Disposal**: Tự động rollback nếu transaction còn active khi dispose

#### 2.2 EF Core UnitOfWork (Inventory & Order Service)

```mermaid
sequenceDiagram
    participant Handler as CommandHandler
    participant UoW as UnitOfWork
    participant Repo as Repository
    participant EF as ApplicationDbContext
    participant DB as Database

    Handler->>UoW: BeginTransactionAsync()
    UoW->>EF: Database.BeginTransactionAsync()
    EF-->>UoW: IDbContextTransaction
    UoW->>UoW: Wrap in DbTransactionAdapter
    UoW-->>Handler: IDbTransaction

    Handler->>Repo: AddAsync(entity)
    Repo->>EF: DbSet.Add(entity)
    
    Handler->>UoW: SaveChangesAsync()
    UoW->>EF: SaveChangesAsync()
    EF->>DB: Execute SQL
    DB-->>EF: Rows affected
    
    Handler->>Handler: transaction.CommitAsync()
    
    alt Exception
        Handler->>Handler: transaction.RollbackAsync()
    end
```

**File**: `Inventory.Infrastructure/UnitOfWork/UnitOfWork.cs` (tương tự cho Order)

```csharp
public class UnitOfWork(
    IInventoryReservationRepository inventoryReservations,
    IInventoryItemRepository inventoryItems,
    // ... other repositories
    ApplicationDbContext context) : IUnitOfWork
{
    public IInventoryReservationRepository InventoryReservations { get; } = inventoryReservations;
    // ... other properties

    public Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        => context.SaveChangesAsync(cancellationToken);

    public async Task<IDbTransaction> BeginTransactionAsync(CancellationToken cancellationToken = default)
    {
        var tx = await context.Database.BeginTransactionAsync(cancellationToken);
        return new DbTransactionAdapter(tx);
    }
}
```

**Giải thích:**
1. **Primary Constructor**: Sử dụng C# 12 primary constructor để inject dependencies
2. **Repository Properties**: Tất cả repositories được inject và expose qua properties
3. **SaveChangesAsync**: Delegate trực tiếp đến EF Core DbContext
4. **BeginTransactionAsync**: Sử dụng EF Core transaction với `DbTransactionAdapter`

### 3. Dependency Injection Registration

```mermaid
flowchart LR
    subgraph Registration["DI Registration Flow"]
        A[Start] --> B[Register DbContext]
        B --> C[Register Interceptors]
        C --> D[Scan Assembly]
        D --> E[Register Repositories]
        E --> F[Register UnitOfWork]
        F --> G[Scoped Lifetime]
    end
```

**Cấu hình DI (Inventory/Order Service):**

```csharp
// Repository & Unit of Work
{
    services.Scan(s => s
        .FromAssemblyOf<InfrastructureMarker>()
        .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
        .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
        .AsImplementedInterfaces()
        .WithScopedLifetime());

    services.AddScoped<Domain.Abstractions.IUnitOfWork, UnitOfWork.UnitOfWork>();
}
```

**Quy trình:**
1. Scrutor scan và register tất cả repositories từ assembly
2. Register UnitOfWork sau repositories để đảm bảo dependencies đã sẵn sàng
3. Cả đều sử dụng **Scoped Lifetime**

**Cấu hình DI (Discount Service):**

```csharp
// Register UnitOfWork as scoped (must be after repositories)
services.AddScoped<Discount.Application.Repositories.IUnitOfWork, Repositories.UnitOfWork>();
```

### 4. Usage Patterns trong Handlers

#### 4.1 Pattern 1: Explicit Transaction

```mermaid
sequenceDiagram
    participant CH as CommandHandler
    participant UoW as IUnitOfWork
    participant Tx as IDbTransaction
    participant Repo as Repository
    participant Ext as ExternalService

    CH->>UoW: BeginTransactionAsync()
    UoW-->>CH: IDbTransaction

    Note over CH: try block starts
    
    CH->>Ext: Validate external data
    Ext-->>CH: Validation result
    
    CH->>Repo: Check existence
    Repo-->>CH: Entity or null
    
    CH->>Repo: AddAsync(entity)
    
    CH->>UoW: SaveChangesAsync()
    CH->>Tx: CommitAsync()
    
    Note over CH: catch block
    alt Exception occurs
        CH->>Tx: RollbackAsync()
        CH-->>CH: throw
    end
```

**Ví dụ: CreateInventoryItemCommand**

```csharp
public async Task<Guid> Handle(CreateInventoryItemCommand command, CancellationToken cancellationToken)
{
    var transaction = await unitOfWork.BeginTransactionAsync(cancellationToken);

    try
    {
        // Validate product exists
        var product = await catalogGrpc.GetProductByIdAsync(dto.ProductId.ToString(), cancellationToken)
            ?? throw new ClientValidationException(MessageCode.ProductIsNotExists, dto.ProductId);
        
        // Check for existing inventory item
        var existingItem = await unitOfWork.InventoryItems
            .FirstOrDefaultAsync(x => x.Product.Id == product.Id && x.LocationId == dto.LocationId, cancellationToken);

        if (existingItem is not null) 
            throw new ClientValidationException(MessageCode.InventoryItemAlreadyExists, dto.ProductId);

        // Create entity
        var entity = InventoryItemEntity.Create(...);
        await unitOfWork.InventoryItems.AddAsync(entity);

        // Persist
        await unitOfWork.SaveChangesAsync(cancellationToken);
        await transaction.CommitAsync(cancellationToken);

        return entity.Id;
    }
    catch (Exception)
    {
        await transaction.RollbackAsync(cancellationToken);
        throw;
    }
}
```

**Đặc điểm:**
- Sử dụng `try-catch-finally` pattern
- Explicit `BeginTransactionAsync`, `CommitAsync`, `RollbackAsync`
- Validate trước khi thực hiện thay đổi
- `SaveChangesAsync` trước khi commit transaction

#### 4.2 Pattern 2: Implicit Transaction

```mermaid
sequenceDiagram
    participant CH as CommandHandler
    participant UoW as IUnitOfWork
    participant EF as DbContext
    participant DB as Database

    Note over CH: No explicit transaction
    
    CH->>UoW: Orders.AddAsync(order)
    UoW->>EF: Track entity
    
    CH->>UoW: SaveChangesAsync()
    UoW->>EF: SaveChangesAsync()
    
    Note right of EF: EF Core auto-creates<br/>implicit transaction
    
    EF->>DB: BEGIN TRANSACTION
    EF->>DB: INSERT/UPDATE/DELETE
    EF->>DB: COMMIT
    DB-->>EF: Success
    EF-->>UoW: Rows affected
    UoW-->>CH: Rows affected
```

**Ví dụ: CreateOrderCommand**

```csharp
public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken cancellationToken)
{
    // Build domain entities
    var order = OrderEntity.Create(...);
    
    // Fetch and validate products
    var products = await catalogGrpc.GetAllAvailableProductsAsync(cancellationToken);
    
    // Add order items
    foreach (var item in dto.OrderItems)
    {
        var product = products.Items.FirstOrDefault(x => x.Id == item.ProductId);
        order.AddOrderItem(product, item.Quantity);
    }
    
    // Apply discount if applicable
    if (!string.IsNullOrWhiteSpace(dto.CouponCode))
    {
        var discount = await discountGrpc.EvaluateCouponAsync(...);
        order.ApplyDiscount(discount);
    }
    
    // Persist - EF Core creates implicit transaction
    await unitOfWork.Orders.AddAsync(order, cancellationToken);
    await unitOfWork.SaveChangesAsync(cancellationToken);

    return order.Id;
}
```

**Đặc điểm:**
- Không sử dụng explicit transaction
- EF Core tự động tạo implicit transaction cho `SaveChangesAsync`
- Phù hợp cho single-operation scenarios
- Code đơn giản hơn, dễ đọc

### 5. So Sánh Các Patterns

```mermaid
flowchart TD
    subgraph Decision["Transaction Decision Flow"]
        Start[Start] --> MultiOp{Multiple Operations?}
        MultiOp -->|Yes| Explicit[Explicit Transaction]
        MultiOp -->|No| External{External Service Calls?}
        External -->|Yes| Explicit
        External -->|No| Critical{Critical Business Logic?}
        Critical -->|Yes| Explicit
        Critical -->|No| Implicit[Implicit Transaction]
    end
    
    Explicit --> ExplicitCode["BeginTransactionAsync<br/>try-catch-finally<br/>Commit/Rollback"]
    Implicit --> ImplicitCode["SaveChangesAsync only<br/>EF Core auto-transaction"]
```

## C. Tích Hợp Outbox/Inbox Pattern

### 1. Kiến Trúc Outbox/Inbox với UnitOfWork

```mermaid
sequenceDiagram
    participant Handler as CommandHandler
    participant UoW as IUnitOfWork
    participant Outbox as OutboxMessages Repo
    participant Business as Business Repo
    participant DB as Database
    participant Worker as BackgroundWorker
    participant Broker as MessageBroker

    Note over Handler: Same Transaction
    
    Handler->>Business: AddAsync(entity)
    Handler->>Outbox: AddAsync(outboxMessage)
    Handler->>UoW: SaveChangesAsync()
    
    UoW->>DB: BEGIN TRANSACTION
    UoW->>DB: INSERT Business Data
    UoW->>DB: INSERT Outbox Message
    UoW->>DB: COMMIT
    
    Note over Worker: Later
    Worker->>DB: Read pending messages
    DB-->>Worker: Messages
    Worker->>Broker: Publish message
    Broker-->>Worker: Ack
    Worker->>DB: Mark as processed
```

### 2. Outbox Pattern

Outbox pattern được sử dụng để đảm bảo message được publish đáng tin cậy:

1. Message được lưu vào `OutboxMessages` table cùng với business data trong cùng một transaction
2. Background worker đọc từ Outbox và publish message
3. Nếu publish thành công, message được đánh dấu đã xử lý

### 3. Inbox Pattern

Inbox pattern ngăn chặn việc xử lý message trùng lặp:

1. Message từ message broker được lưu vào `InboxMessages`
2. Xử lý message và đánh dấu đã xử lý
3. Nếu message đã tồn tại, bỏ qua xử lý

## D. So Sánh Các Triển Khai

| Feature | Discount (MongoDB) | Inventory (EF Core) | Order (EF Core) |
|---------|-------------------|---------------------|-----------------|
| **Database** | MongoDB | SQL Server/PostgreSQL/MySQL | SQL Server/PostgreSQL/MySQL |
| **Transaction Type** | MongoDB Session | EF Core Transaction | EF Core Transaction |
| **Disposal** | IDisposable, IAsyncDisposable | Không (repositories scoped) | Không (repositories scoped) |
| **Outbox/Inbox** | Không | Có | Có |
| **SaveChanges** | Manual commit hoặc auto | EF Core Change Tracker | EF Core Change Tracker |
| **Error Handling** | Rollback session | Rollback transaction | Rollback transaction |

## E. Best Practices

### 1. Luôn Sử Dụng Transaction cho Multi-Operation

```mermaid
flowchart TD
    subgraph Good["✓ CORRECT"]
        A[BeginTransaction] --> B[Operation 1]
        B --> C[Operation 2]
        C --> D[SaveChanges]
        D --> E[Commit]
        E --> F[Success]
        D -. Exception .-> G[Rollback]
    end
    
    subgraph Bad["✗ WRONG"]
        H[Operation 1] --> I[SaveChanges]
        I --> J[Operation 2]
        J --> K[SaveChanges]
        K --> L[Partial Success!]
    end
```

### 2. Code Pattern Tốt

**Đúng:**
```csharp
await unitOfWork.Repo1.AddAsync(entity1);
await unitOfWork.Repo2.AddAsync(entity2);
await unitOfWork.SaveChangesAsync(ct);  // Single SaveChanges
```

**Sai:**
```csharp
await unitOfWork.Repo1.AddAsync(entity1);
await unitOfWork.SaveChangesAsync(ct);  // Multiple SaveChanges
await unitOfWork.Repo2.AddAsync(entity2);
await unitOfWork.SaveChangesAsync(ct);  // Tạo nhiều transaction
```

### 3. Exception Handling Pattern

```mermaid
flowchart TD
    A[BeginTransaction] --> B{try}
    B --> C[Business Logic]
    C --> D[SaveChanges]
    D --> E[Commit]
    E --> F[Return Result]
    
    C -. Exception .-> G{catch}
    D -. Exception .-> G
    E -. Exception .-> G
    G --> H[Rollback]
    H --> I[Throw Exception]
```

### 4. Quy Tắc Quan Trọng

| Quy Tắc | Mô Tả |
|---------|-------|
| **Single SaveChanges** | Chỉ gọi `SaveChangesAsync` một lần cho tất cả thay đổi |
| **Always Rollback** | Luôn rollback khi exception xảy ra |
| **Validate First** | Validate input trước khi bắt đầu transaction |
| **Keep Transactions Short** | Không thực hiện operations dài trong transaction |
| **External Calls** | Gọi external services trước khi bắt đầu transaction nếu có thể |

## F. Tools và Technologies

| Category | Technologies |
|----------|-------------|
| **Ngôn ngữ** | C# 12 (.NET 8) |
| **MongoDB** | MongoDB.Driver, MongoDB Transactions |
| **EF Core** | Entity Framework Core 8, SQL Server/PostgreSQL/MySQL providers |
| **DI Container** | Microsoft.Extensions.DependencyInjection, Scrutor |
| **Architecture** | Clean Architecture, Domain-Driven Design |

## G. Kết Luận

Hệ thống Unit of Work trong ProgCoder Shop Microservices được thiết kế linh hoạt, hỗ trợ cả MongoDB và EF Core với các patterns:

1. **Repository Pattern**: Tách biệt data access logic
2. **Unit of Work Pattern**: Quản lý transactions
3. **Outbox/Inbox Pattern**: Đảm bảo reliable messaging

Hệ thống này đảm bảo tính nhất quán của dữ liệu trong microservices architecture với event-driven communication.

```mermaid
flowchart TB
    subgraph Summary["Unit of Work System Summary"]
        direction TB
        
        A[Unit of Work Pattern] --> B[MongoDB Implementation]
        A --> C[EF Core Implementation]
        
        B --> D[Session-based<br/>Transactions]
        C --> E[DbContext<br/>Transactions]
        
        D --> F[Discount Service]
        E --> G[Inventory Service]
        E --> H[Order Service]
        
        G --> I[Outbox/Inbox<br/>Pattern]
        H --> I
    end
```
