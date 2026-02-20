# EF Core Transaction Flow trong Basket Checkout Process

## Mục lục

1. [Tổng quan kiến trúc](#tổng-quan-kiến-trúc)
2. [Chi tiết từng phase](#chi-tiết-từng-phase)
3. [Sơ đồ luồng hoạt động](#sơ-đồ-luồng-hoạt-động)
4. [EF Core Internals](#ef-core-internals)
5. [Domain Event Dispatch Timing](#domain-event-dispatch-timing)
6. [Tóm tắt các hàm chính](#tóm-tắt-các-hàm-chính)
7. [Câu hỏi thường gặp](#câu-hỏi-thường-gặp)

---

## Tổng quan kiến trúc

Dự án ProgCoder Shop Microservices sử dụng kiến trúc Event-Driven với Outbox pattern để đảm bảo tính nhất quán dữ liệu giữa các microservices. Luồng checkout trong Basket Service thể hiện rõ cách EF Core quản lý transaction để đảm bảo rằng tất cả các thao tác database (INSERT, UPDATE) và Outbox messages đều được commit cùng nhau hoặc rollback cùng nhau.

Transaction trong EF Core là một khái niệm quan trọng đảm bảo tính Atomicity trong nguyên tắc ACID. Khi thực hiện checkout, hệ thống cần đảm bảo rằng đơn hàng được tạo, giỏ hàng được cập nhật trạng thái, và message outbox được thêm vào database - tất cả phải thành công hoặc không có gì xảy ra. Nếu bất kỳ bước nào thất bại, toàn bộ transaction sẽ được rollback và không có dữ liệu nào được ghi vào database.

Kiến trúc này đặc biệt quan trọng trong môi trường distributed systems nơi mà việc đảm bảo tính nhất quán dữ liệu giữa các services là thách thức lớn. Outbox pattern giúp giải quyết vấn đề này bằng cách lưu trữ messages trong cùng database với business data, đảm bảo rằng messages chỉ được publish khi và chỉ khi business transaction thành công.

---

## Chi tiết từng phase

### Phase 1: Mở Transaction (Open Transaction)

Giai đoạn đầu tiên trong luồng checkout là mở một database transaction. Điều này được thực hiện thông qua phương thức `OpenTransactionAsync()` trong `BasketRepository`. Khi transaction được mở, EF Core sẽ thiết lập một kết nối đến database và bắt đầu một transaction context mà tất cả các thao tác tiếp theo sẽ được thực hiện trong đó cho đến khi được commit hoặc rollback.

Việc mở transaction sớm trong quy trình là cần thiết để đảm bảo rằng tất cả các thay đổi data có thể được nhóm lại và commit cùng nhau. Nếu chúng ta không mở transaction một cách tường minh, EF Core sẽ tự động mở transaction cho mỗi lệnh SQL, điều này có thể dẫn đến tình trạng không nhất quán dữ liệu nếu một phần của quy trình thất bại.

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Mở một transaction mới hoặc trả về transaction hiện tại
/// </summary>
/// <param name="ct">Cancellation token</param>
/// <returns>DbContextTransaction instance</returns>
public async Task<DbContextTransaction> OpenTransactionAsync(CancellationToken ct = default)
{
    var dbContext = GetDbContext();
    
    // Kiểm tra xem đã có transaction đang mở chưa
    if (dbContext.Database.CurrentTransaction != null)
    {
        _logger.LogInformation("Transaction already exists, returning current transaction");
        return dbContext.Database.CurrentTransaction;
    }
    
    _logger.LogInformation("Opening new database transaction");
    
    // Mở transaction mới
    var transaction = await dbContext.Database.BeginTransactionAsync(ct);
    _transaction = transaction;
    
    _logger.LogInformation("Transaction opened successfully with ID: {TransactionId}", 
        transaction.TransactionId);
    
    return _transaction;
}
```

EF Core Internals bên trong `BeginTransactionAsync()` thực hiện các bước sau: đầu tiên, nó lấy relational connection từ DI container, sau đó mở kết nối đến database nếu chưa mở, tiếp theo gọi phương thức `BeginTransaction()` trên connection của database provider (trong trường hợp này là Npgsql cho PostgreSQL), và cuối cùng wrap transaction trong đối tượng `DbContextTransaction` để quản lý lifecycle.

```csharp
// EF Core Internal Implementation (Simplified)
public async Task<IDbContextTransaction> BeginTransactionAsync(
    CancellationToken cancellationToken = default)
{
    // 1. Lấy relational connection từ DI container
    var connection = _connectionServices.GetRequiredService<IRelationalConnection>();
    
    // 2. Mở kết nối đến database
    await connection.OpenAsync(cancellationToken);
    
    // 3. Gọi native transaction begin của provider
    var nativeTransaction = await connection
        .DbConnection
        .BeginTransactionAsync(cancellationToken);
    
    // 4. Wrap trong DbContextTransaction
    return new DbContextTransaction(
        connection,
        nativeTransaction,
        _transactionManager);
}
```

### Phase 2: Thêm Data vào Transaction

Sau khi transaction được mở, các thao tác thêm và cập nhật dữ liệu sẽ được thực hiện. Điều quan trọng cần hiểu là ở giai đoạn này, dữ liệu chỉ được EF Core track trong ChangeTracker và chưa thực sự được ghi vào database. Tất cả các thay đổi được tích lũy trong bộ nhớ và chờ đợi cho đến khi `SaveChanges()` được gọi.

EF Core sử dụng ChangeTracker để theo dõi trạng thái của tất cả các entities. Mỗi khi bạn gọi `Add()`, `Update()`, hoặc `Remove()`, EF Core sẽ thay đổi trạng thái của entity trong ChangeTracker. Các trạng thái bao gồm: `Added` (entity mới sẽ được INSERT), `Modified` (entity đã tồn tại sẽ được UPDATE), `Deleted` (entity sẽ được DELETE), và `Unchanged` (entity không có thay đổi).

Trong luồng checkout, có ba loại thao tác chính được thực hiện: tạo mới Order từ basket data, cập nhật trạng thái của Basket thành "CheckedOut", và thêm OutboxMessage vào database để chuẩn bị publish event. Tất cả các thao tác này đều được track trong cùng một ChangeTracker và sẽ được thực hiện trong cùng một transaction.

**Thêm Order mới:**

```csharp
// File: src/Services/Basket/Core/Basket.Application/Features/Basket/Commands/BasketCheckoutCommand.cs

/// <summary>
/// Tạo ShoppingCart entity từ basket data
/// </summary>
public static ShoppingCartEntity Create(
    Guid id,
    Guid customerId,
    string? customerName,
    CustomerDomainEvent customer,
    List<CartItem> items,
    List<Coupon> coupons,
    string? promotionCodes,
    Guid transactionId,
    DateTimeOffset? expiresAt)
{
    var entity = new ShoppingCartEntity
    {
        Id = id,
        CustomerId = customerId,
        CustomerName = customerName ?? customer.Name,
        Status = ShoppingCartStatus.Created,
        TransactionId = transactionId,
        ExpiresAt = expiresAt ?? DateTimeOffset.UtcNow.AddHours(configuration["Basket:ExpiresHours"]!),
        Items = items.ConvertAll(x => new CartItemEntity
        {
            Id = Guid.NewGuid(),
            ProductId = x.ProductId,
            ProductName = x.ProductName,
            ProductImage = x.ProductImage,
            UnitPrice = x.UnitPrice,
            Quantity = x.Quantity,
            DiscountRate = x.DiscountRate,
            DiscountPrice = x.DiscountPrice,
            FinalPrice = x.FinalPrice,
            CurrencyCode = x.CurrencyCode
        }),
        Coupons = coupons.ConvertAll(x => new CouponEntity
        {
            Id = Guid.NewGuid(),
            Code = x.Code,
            Name = x.Name,
            Description = x.Description,
            DiscountType = x.DiscountType,
            DiscountValue = x.DiscountValue
        })
    };
    
    return entity;
}
```

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Thêm entity mới vào database
/// </summary>
/// <param name="entity">Entity cần thêm</param>
/// <param name="ct">Cancellation token</param>
public async Task Add(ShoppingCartEntity entity, CancellationToken ct = default)
{
    var dbContext = GetDbContext();
    
    _logger.LogDebug("Adding entity {EntityType} with ID {EntityId} to change tracker",
        entity.GetType().Name, entity.Id);
    
    // EF Core AddAsync - entity được track với state = Added
    await dbContext.AddAsync(entity, ct);
    
    // Entity state trong ChangeTracker: Added
    // Chờ SaveChanges() để thực hiện INSERT
}
```

**Cập nhật Basket Status:**

```csharp
// File: src/Services/Basket/Core/Basket.Domain/Entities/ShoppingCartEntity.cs

/// <summary>
/// Cập nhật trạng thái basket khi checkout
/// </summary>
/// <param name="transactionId">Transaction ID từ payment gateway</param>
public void Checkout(Guid transactionId)
{
    // Validate: Basket phải có items để checkout
    if (Items == null || !Items.Any())
        throw new DomainException(MessageCode.BasketIsEmpty);
    
    // Validate: Basket phải ở trạng thái Created
    if (Status != ShoppingCartStatus.Created)
        throw new DomainException(MessageCode.BasketCannotBeCheckedOut);
    
    Status = ShoppingCartStatus.CheckedOut;
    TransactionId = transactionId;
    
    // Thêm Domain Event để trigger các side effects
    AddDomainEvent(new BasketCheckoutDomainEvent(this));
    
    _logger.LogInformation("Basket {BasketId} checked out with transaction {TransactionId}",
        Id, transactionId);
}
```

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Cập nhật entity hiện có
/// </summary>
/// <param name="entity">Entity cần cập nhật</param>
/// <param name="ct">Cancellation token</param>
public async Task Update(ShoppingCartEntity entity, CancellationToken ct = default)
{
    var dbContext = GetDbContext();
    
    _logger.LogDebug("Updating entity {EntityType} with ID {EntityId} in change tracker",
        entity.GetType().Name, entity.Id);
    
    // EF Core Update - entity được track với state = Modified
    dbContext.Update(entity);
    
    // Entity state trong ChangeTracker: Modified
    // Chờ SaveChanges() để thực hiện UPDATE
}
```

**Thêm Outbox Message:**

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/OutboxRepository.cs

/// <summary>
/// Thêm message vào Outbox table
/// </summary>
/// <param name="message">Outbox message entity</param>
/// <param name="ct">Cancellation token</param>
/// <returns>True nếu thêm thành công</returns>
public async Task<bool> AddMessageAsync(OutboxMessageEntity message, CancellationToken ct = default)
{
    var dbContext = GetDbContext();
    
    _logger.LogInformation("Adding message {MessageId} to outbox with type {EventType}",
        message.Id, message.EventType);
    
    await dbContext.AddAsync(message, ct);
    
    // OutboxMessage state trong ChangeTracker: Added
    // Sẽ được INSERT cùng lúc với Order và Basket update
    
    _logger.LogDebug("Message added to change tracker, awaiting SaveChanges");
    
    return true;
}
```

**Trạng thái ChangeTracker sau khi thêm data:**

```
ChangeTracker.Entries() sau khi thêm data:

| Entity Type        | State    | Operation | Trong Transaction? |
|--------------------|----------|-----------|-------------------|
| ShoppingCartEntity | Added    | INSERT    | ✅ Có             |
| CartItemEntity[]   | Added    | INSERT    | ✅ Có             |
| CouponEntity[]     | Added    | INSERT    | ✅ Có             |
| ShoppingCartEntity | Modified | UPDATE    | ✅ Có             |
| OutboxMessageEntity| Added    | INSERT    | ✅ Có             |
```

### Phase 3: SaveChanges

Giai đoạn `SaveChanges()` là nơi EF Core thực sự tương tác với database. Phương thức này thực hiện nhiều bước quan trọng: phát hiện thay đổi, tạo SQL commands, thực thi SQL trong transaction hiện tại, và cuối cùng là dispatch domain events nếu có interceptor được cấu hình.

Quan trọng nhất, `SaveChanges()` thực thi tất cả các SQL commands TRONG TRANSACTION đã mở. Điều này có nghĩa là nếu bất kỳ SQL command nào thất bại (ví dụ: constraint violation, connection timeout), toàn bộ transaction sẽ được rollback và không có dữ liệu nào được ghi vào database. Đây là cách EF Core đảm bảo tính atomicity của transaction.

Sau khi SQL được thực thi thành công, EF Core sẽ cập nhật các entities với database-generated values (như identity columns, timestamps, v.v.). Điều này đảm bảo rằng entities trong memory có thông tin đồng bộ với database, đặc biệt quan trọng cho các generated IDs.

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Lưu tất cả thay đổi vào database
/// </summary>
/// <param name="ct">Cancellation token</param>
public async Task SaveChangesAsync(CancellationToken ct = default)
{
    var dbContext = GetDbContext();
    
    _logger.LogDebug("Calling SaveChanges on DbContext");
    
    try
    {
        // EF Core sẽ:
        // 1. Detect changes trong ChangeTracker
        // 2. Generate SQL commands
        // 3. Execute SQL trong current transaction
        // 4. Dispatch domain events (nếu có interceptor)
        // 5. Update entities với database values
        await dbContext.SaveChangesAsync(ct);
        
        _logger.LogInformation("SaveChanges completed successfully");
    }
    catch (DbUpdateConcurrencyException ex)
    {
        _logger.LogError(ex, "Concurrency conflict during SaveChanges");
        throw;
    }
    catch (DbUpdateException ex)
    {
        _logger.LogError(ex, "Database update error during SaveChanges");
        throw;
    }
}
```

**EF Core Internals - SaveChangesAsync:**

Để hiểu rõ hơn về cách EF Core xử lý SaveChanges, chúng ta hãy xem xét các bước nội bộ mà EF Core thực hiện. Quá trình này bao gồm việc phát hiện thay đổi trong ChangeTracker, xác thực trạng thái của các entities, tạo các SQL commands phù hợp, thực thi các commands trong transaction, và cuối cùng là dispatch domain events.

```csharp
// Microsoft.EntityFrameworkCore.ChangeTracking.Internal (Simplified)
public virtual async Task<int> SaveChangesAsync(
    bool acceptAllChangesOnSuccess = true,
    CancellationToken cancellationToken = new CancellationToken())
{
    // === Bước 1: Detect Changes ===
    // Quét ChangeTracker để tìm các entities có thay đổi
    ChangeTracker.DetectChanges();
    
    var entries = ChangeTracker.Entries()
        .Where(e => e.State != EntityState.Unchanged)
        .ToArray();
    
    _logger.LogDebug("Detected {Count} entities with changes", entries.Length);
    
    // === Bước 2: Validate State ===
    // Kiểm tra các ràng buộc và validate entities
    foreach (var entry in entries)
    {
        if (entry.State == EntityState.Added)
        {
            ValidateAddedEntry(entry);
        }
        else if (entry.State == EntityState.Modified)
        {
            ValidateModifiedEntry(entry);
        }
    }
    
    // === Bước 3: Generate SQL Commands ===
    // Tạo các SQL commands từ entity changes
    var commandBatches = await _queryGenerator.GenerateCommandsAsync(entries);
    
    _logger.LogDebug("Generated {Count} command batches", commandBatches.Count);
    
    // === Bước 4: Execute SQL in Transaction ===
    // Thực thi tất cả SQL commands trong transaction hiện tại
    foreach (var batch in commandBatches)
    {
        try
        {
            await batch.ExecuteNonQueryAsync(cancellationToken);
            _logger.LogDebug("Executed SQL batch successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "SQL execution failed");
            throw;
        }
    }
    
    // === Bước 5: Update Entity with Database Values ===
    // Cập nhật entities với các giá trị từ database (IDs, timestamps, etc.)
    foreach (var entry in entries)
    {
        entry.PopulateFromDatabaseValues();
    }
    
    // === Bước 6: Dispatch Domain Events (nếu có Interceptor) ===
    // Gọi các domain event handlers
    await DispatchDomainEvents();
    
    // === Bước 7: Accept All Changes ===
    // Đánh dấu tất cả entities là Unchanged
    if (acceptAllChangesOnSuccess)
    {
        ChangeTracker.AcceptAllChanges();
    }
    
    return entries.Length;
}
```

**SQL Generated Example:**

Dưới đây là ví dụ về các SQL commands được tạo ra và thực thi trong transaction. Mỗi command tương ứng với một thay đổi trong ChangeTracker và tất cả đều được thực thi trong cùng một transaction database.

```sql
-- Transaction SQL Execution
BEGIN TRANSACTION;

-- INSERT Order (từ Basket data)
INSERT INTO "Orders" (
    "Id", 
    "CustomerId", 
    "CustomerName", 
    "Status", 
    "TransactionId", 
    "ExpiresAt",
    "CreatedAt", 
    "ModifiedAt"
)
VALUES (
    'a1b2c3d4-e5f6-7890-abcd-ef1234567890', 
    'customer-123', 
    'John Doe', 
    'Created', 
    'txn-456', 
    '2025-02-15 10:30:00+00',
    NOW(), 
    NOW()
);

-- UPDATE Basket status
UPDATE "Baskets" 
SET 
    "Status" = 'CheckedOut', 
    "TransactionId" = 'txn-456',
    "ModifiedAt" = NOW()
WHERE "Id" = 'basket-789';

-- INSERT OutboxMessage (BasketCheckoutIntegrationEvent)
INSERT INTO "OutboxMessages" (
    "Id",
    "EventType",
    "Content",
    "OccurredOnUtc",
    "ProcessedOnUtc",
    "AttemptCount",
    "MaxAttempts"
)
VALUES (
    'outbox-msg-001',
    'BasketCheckoutIntegrationEvent',
    '{
        "Id": "evt-001",
        "BasketId": "basket-789",
        "Customer": {
            "Id": "customer-123",
            "Name": "John Doe",
            "Email": "john@example.com"
        },
        "Items": [...],
        "OccurredOn": "2025-02-15T10:30:00Z"
    }',
    NOW(),  -- OccurredOnUtc
    NULL,   -- ProcessedOnUtc (chưa processed)
    0,      -- AttemptCount
    3       -- MaxAttempts
);

COMMIT TRANSACTION;
```

### Phase 4: Commit Transaction

Sau khi `SaveChangesAsync()` hoàn tất thành công, transaction cần được commit. Việc commit là thao tác cuối cùng trong quy trình transaction, xác nhận rằng tất cả các thay đổi đã được thực hiện và nên được persist vĩnh viễn vào database.

Khi `CommitAsync()` được gọi, EF Core sẽ thực hiện lệnh `COMMIT` trên database transaction. Điều này yêu cầu database ghi các thay đổi từ transaction log vào data files một cách vĩnh viễn. Nếu commit thành công, tất cả các thay đổi sẽ được nhìn thấy bởi các transactions khác. Nếu commit thất bại (ví dụ: do crash), database sẽ tự động rollback transaction khi khởi động lại.

Một điểm quan trọng cần lưu ý là việc commit chỉ thành công nếu `SaveChangesAsync()` đã hoàn tất mà không có lỗi. Nếu có bất kỳ lỗi nào trong quá trình save, transaction sẽ được rollback tự động bởi EF Core hoặc explicit trong catch block.

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Commit transaction hiện tại
/// </summary>
/// <param name="transaction">Transaction instance</param>
/// <param name="ct">Cancellation token</param>
public async Task CommitAsync(DbContextTransaction transaction, CancellationToken ct = default)
{
    try
    {
        var dbContext = GetDbContext();
        
        _logger.LogInformation("Committing transaction {TransactionId}", 
            transaction.TransactionId);
        
        // EF Core CommitTransaction - thực thi COMMIT SQL
        await dbContext.Database.CommitTransactionAsync(ct);
        
        _logger.LogInformation("Transaction {TransactionId} committed successfully", 
            transaction.TransactionId);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to commit transaction, initiating rollback");
        
        // Rollback nếu commit thất bại
        await RollbackAsync(transaction, ct);
        
        throw;
    }
}
```

**EF Core Internals - CommitTransactionAsync:**

```csharp
// Microsoft.EntityFrameworkCore.RelationalDatabase
public virtual async Task CommitTransactionAsync(CancellationToken cancellationToken = default)
{
    var transaction = CurrentTransaction;
    if (transaction == null)
    {
        throw new InvalidOperationException("No active transaction to commit");
    }
    
    try
    {
        // Gọi commit trên native transaction
        await transaction.CommitAsync(cancellationToken);
        
        _logger.LogDebug("Transaction committed successfully");
        
        // Xóa transaction reference khỏi DbContext
        ClearTransaction();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to commit transaction");
        throw;
    }
}
```

**Npgsql (PostgreSQL) Implementation:**

```csharp
// Npgsql (Simplified)
public override async Task CommitAsync(CancellationToken cancellationToken = default)
{
    // Tạo và thực thi COMMIT command
    var command = new NpgsqlCommand("COMMIT;");
    await command.ExecuteNonQueryAsync(cancellationToken);
    
    // Connection state chuyển từ InTransaction -> None
    // Transaction đã được persist vào database
    
    _logger.LogInformation("PostgreSQL transaction committed");
}
```

### Phase 5: Rollback Transaction (Error Handling)

Trong trường hợp có lỗi xảy ra trong quá trình checkout, transaction cần được rollback để đảm bảo không có dữ liệu không nhất quán được ghi vào database. Việc rollback phải được thực hiện trong finally block hoặc catch block để đảm bảo luôn được gọi ngay cả khi có exception.

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Rollback transaction khi có lỗi
/// </summary>
/// <param name="transaction">Transaction instance</param>
/// <param name="ct">Cancellation token</param>
public async Task RollbackAsync(DbContextTransaction transaction, CancellationToken ct = default)
{
    try
    {
        var dbContext = GetDbContext();
        
        _logger.LogWarning("Rolling back transaction {TransactionId}", 
            transaction.TransactionId);
        
        // EF Core RollbackTransaction - thực thi ROLLBACK SQL
        await dbContext.Database.RollbackTransactionAsync(ct);
        
        _logger.LogInformation("Transaction {TransactionId} rolled back successfully", 
            transaction.TransactionId);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to rollback transaction properly");
        
        // Nếu rollback cũng thất bại, transaction sẽ tự động 
        // được cleanup bởi database connection dispose
    }
}
```

### Phase 6: Dispose Transaction

Sau khi transaction hoàn tất (commit hoặc rollback), tài nguyên cần được giải phóng. Điều này bao gồm đóng connection đến database và làm sạch transaction object để garbage collector có thể thu hồi bộ nhớ.

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs

/// <summary>
/// Dispose transaction và cleanup resources
/// </summary>
public async ValueTask DisposeAsync()
{
    await DisposeAsyncCore();
    
    GC.SuppressFinalize(this);
}

/// <summary>
/// Dispose async core logic
/// </summary>
protected virtual async ValueTask DisposeAsyncCore()
{
    if (_transaction != null)
    {
        _logger.LogDebug("Disposing transaction {TransactionId}", 
            _transaction.TransactionId);
        
        await _transaction.DisposeAsync();
        _transaction = null;
        
        _logger.LogInformation("Transaction disposed successfully");
    }
}
```

**EF Core Internals - DisposeAsync:**

```csharp
// Microsoft.EntityFrameworkCore.Storage.IDbContextTransaction
public virtual async ValueTask DisposeAsync()
{
    try
    {
        // Đóng connection đến database
        if (Connection.State == ConnectionState.Open)
        {
            await Connection.CloseAsync();
        }
        
        // Transaction object sẵn sàng cho garbage collection
        
        _logger.LogDebug("Transaction resources released");
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error during transaction dispose");
        // Không throw trong dispose để tránh shadow exception
    }
}
```

---

## Sơ đồ luồng hoạt động

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    EF CORE TRANSACTION LIFECYCLE                                         │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │  1. REQUEST PHASE                                                               │   │
│  │  ┌────────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │  Carter Endpoint: POST /basket/checkout                                     │  │   │
│  │  │  ↓                                                                        │  │   │
│  │  │  CarterHelper.CallAsync(BasketCheckoutCommand)                              │  │   │
│  │  │  ↓                                                                        │  │   │
│  │  │  MediatR dispatches to BasketCheckoutCommandHandler                         │  │   │
│  │  └────────────────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────────────┘   │
│                                        ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │  2. TRANSACTION OPEN PHASE (DbContextTransaction)                               │   │
│  │  ┌────────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │  BasketCheckoutCommandHandler                                              │  │   │
│  │  │  │                                                                          │  │   │
│  │  │  │  using var transaction = await _basketRepository                        │  │   │
│  │  │  │      .OpenTransactionAsync(cancellationToken);  ◄── MỞ TRANSACTION     │  │   │
│  │  │  │                                                                          │  │   │
│  │  │  │  EF Core Implementation:                                                │  │   │
│  │  │  │  _dbContext.Database.BeginTransaction()                                  │  │   │
│  │  │  │       ↓                                                                  │  │   │
│  │  │  │  _transaction = dbContext.Database.CurrentTransaction                   │  │   │
│  │  │  └──────────────────────────────────────────────────────────────────────────┘  │   │
│  │  └──────────────────────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                                        │
│  │                              ▼                                                        │
│  │  ┌────────────────────────────────────────────────────────────────────────────┐      │
│  │  │  3. DATA MODIFICATION PHASE (Add to Transaction)                           │      │
│  │  │  ┌──────────────────────────────────────────────────────────────────────┐  │      │
│  │  │  │  a) CREATE ORDER (if basket has items)                                 │  │      │
│  │  │  │     ┌─────────────────────────────────────────────────────────────┐   │  │      │
│  │  │  │     │  var order = ShoppingCartEntity.Create(...)                  │   │  │      │
│  │  │  │     │  await _basketRepository.Add(order, ct);  ◄── TRACKING       │   │  │      │
│  │  │  │     │  _dbContext.ChangeTracker.Entries() → Added state             │   │  │      │
│  │  │  │     └─────────────────────────────────────────────────────────────┘   │  │      │
│  │  │  │                                                                │        │  │      │
│  │  │  │  b) UPDATE BASKET STATUS                                        │        │  │      │
│  │  │  │     ┌─────────────────────────────────────────────────────────────┐   │  │      │
│  │  │  │     │  basket.Checkout(out transactionId);                       │   │  │      │
│  │  │  │     │  await _basketRepository.Update(basket, ct);  ◄── TRACKING │   │  │      │
│  │  │  │     │  _dbContext.ChangeTracker.Entries() → Modified state        │   │  │      │
│  │  │  │     └─────────────────────────────────────────────────────────────┘   │  │      │
│  │  │  │                                                                │        │  │      │
│  │  │  │  c) ADD OUTBOX MESSAGE (Domain Event Handler)                   │        │  │      │
│  │  │  │     ┌─────────────────────────────────────────────────────────────┐   │  │      │
│  │  │  │     │  BasketCheckoutDomainEventHandler.Handle()                 │   │  │      │
│  │  │  │     │  await outboxRepo.AddMessageAsync(outboxMessage, ct)       │   │  │      │
│  │  │  │     │  _dbContext.ChangeTracker.Entries() → Added state           │   │  │      │
│  │  │  │     └─────────────────────────────────────────────────────────────┘   │  │      │
│  │  │  │                                                                │        │  │      │
│  │  │  │  ⚠️  TẠI ĐÂY: Tất cả entities đều trong                         │        │  │      │
│  │  │  │      _dbContext.ChangeTracker với state = Added/Modified         │        │  │      │
│  │  │  │      NHƯNG CHƯA được commit vào database                        │        │  │      │
│  │  │  └──────────────────────────────────────────────────────────────────┘  │      │
│  │  └─────────────────────────────────────────────────────────────────────────┘      │
│  │                              │                                                        │
│  │                              ▼                                                        │
│  │  ┌────────────────────────────────────────────────────────────────────────────┐      │
│  │  │  4. SAVE CHANGES PHASE                                                     │      │
│  │  │  ┌──────────────────────────────────────────────────────────────────────┐  │      │
│  │  │  │  try                                                                      │  │   │
│  │  │  │  {                                                                         │  │   │
│  │  │  │      await _basketRepository.SaveChangesAsync(cancellationToken);        │  │   │
│  │  │  │                                                                        │  │   │
│  │  │  │      EF Core Internals:                                                 │  │   │
│  │  │  │      1. _dbContext.SaveChanges()                                       │  │   │
│  │  │  │         ↓                                                               │  │   │
│  │  │  │      2. ChangeTracker.DetectChanges()                                  │  │   │
│  │  │  │         ↓                                                               │  │   │
│  │  │  │      3. Generate SQL INSERT/UPDATE statements                          │  │   │
│  │  │  │         ↓                                                               │  │   │
│  │  │  │      4. Execute SQL trong TRANSACTION                                  │  │   │
│  │  │  │         ↓                                                               │  │   │
│  │  │  │      5. Update entity IDs from database                                │  │   │
│  │  │  │         ↓                                                               │  │   │
│  │  │  │      6. Dispatch Domain Events (Interceptor)                           │  │   │
│  │  │  │  }                                                                         │  │   │
│  │  │  └──────────────────────────────────────────────────────────────────────────┘  │
│  │  │                              │                                                   │
│  │  │                              ▼                                                   │
│  │  │  ┌────────────────────────────────────────────────────────────────────────┐   │
│  │  │  │  5. TRANSACTION COMMIT PHASE                                           │   │
│  │  │  │  ┌────────────────────────────────────────────────────────────────────┐│   │
│  │  │  │  │  await transaction.CommitAsync(ct);  ◄── COMMIT                    ││   │
│  │  │  │  │                                                               │     ││   │
│  │  │  │  │  EF Core Implementation:                                         │     ││   │
│  │  │  │  │  _dbContext.Database.Commit()                                    │     ││   │
│  │  │  │  │       ↓                                                           │     ││   │
│  │  │  │  │  _transaction.Commit()  (NpgsqlTransaction.Commit())             │     ││   │
│  │  │  │  │       ↓                                                           │     ││   │
│  │  │  │  │  PostgreSQL: COMMIT;                                             │     ││   │
│  │  │  │  │       ↓                                                           │     ││   │
│  │  │  │  │  Tất cả changes được PERSIST vào database                        │     ││   │
│  │  │  │  │       • INSERT Order                                              │     ││   │
│  │  │  │  │       • UPDATE Basket                                             │     ││   │
│  │  │  │  │       • INSERT OutboxMessage                                      │     ││   │
│  │  │  │  └────────────────────────────────────────────────────────────────────┘│   │
│  │  │  └─────────────────────────────────────────────────────────────────────────┘   │
│  │  │                              │                                                    │
│  │  │                              ▼                                                    │
│  │  │  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │  │  │  6. TRANSACTION DISPOSE PHASE                                          │    │
│  │  │  │  ┌────────────────────────────────────────────────────────────────────┐│    │
│  │  │  │  │  await transaction.DisposeAsync();  ◄── DISPOSE                    ││    │
│  │  │  │  │                                                               │     ││    │
│  │  │  │  │  EF Core Implementation:                                         │     ││    │
│  │  │  │  │  _transaction.Dispose()                                          │     ││    │
│  │  │  │  │       ↓                                                           │     ││    │
│  │  │  │  │  PostgreSQL: Connection returned to pool                          │     ││    │
│  │  │  │  │       ↓                                                           │     ││    │
│  │  │  │  │  Transaction object ready for garbage collection                  │     ││    │
│  │  │  │  └────────────────────────────────────────────────────────────────────┘│    │
│  │  │  └─────────────────────────────────────────────────────────────────────────┘    │
│  │  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## EF Core Internals

### ChangeTracker Mechanics

EF Core ChangeTracker là thành phần quan trọng quản lý trạng thái của các entities. Mỗi khi bạn thêm, cập nhật, hoặc xóa một entity, ChangeTracker sẽ theo dõi các thay đổi này cho đến khi `SaveChanges()` được gọi. Cơ chế này cho phép EF Core tạo ra các SQL commands tối ưu và thực hiện chúng trong một transaction.

ChangeTracker hoạt động dựa trên concept của "snapshot" - khi một entity được query từ database hoặc thêm vào context, một snapshot của trạng thái ban đầu được lưu trữ. Khi entity có thay đổi, EF Core so sánh trạng thái hiện tại với snapshot để xác định những gì cần được update trong database.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           CHANGE TRACKER MECHANICS                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │  TRẠNG THÁI CỦA ENTITIES                                                        │   │
│  │                                                                                  │   │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │   │
│  │  │   Added     │   │  Modified   │   │   Deleted   │   │ Unchanged   │          │   │
│  │  │             │   │             │   │             │   │             │          │   │
│  │  │ New entity  │   │ Existing    │   │ Existing    │   │ No changes  │          │   │
│  │  │ will be     │   │ entity with │   │ entity to   │   │ since query │          │   │
│  │  │ INSERTed    │   │ changes will│   │ be DELETED  │   │ or attach   │          │   │
│  │  │             │   │ be UPDATEd  │   │             │   │             │          │   │
│  │  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘          │   │
│  │                                                                                  │   │
│  └──────────────────────────────────────────────────────────────────────────────────┘   │
│                                        ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │  LIFE CYCLE TRONG BASKET CHECKOUT                                                 │   │
│  │                                                                                  │   │
│  │  1. Query Basket (Unchanged)                                                      │   │
│  │     ┌────────────────────────────────────────────────────────────────────────┐   │   │
│  │     │  var basket = await repository.GetByIdAsync(basketId);                 │   │   │
│  │     │  ChangeTracker: [Basket: Unchanged]                                     │   │   │
│  │     └────────────────────────────────────────────────────────────────────────┘   │   │
│  │                                        ↓                                          │   │
│  │  2. Create Order (Added)                                                         │   │
│  │     ┌────────────────────────────────────────────────────────────────────────┐   │   │
│  │     │  var order = ShoppingCartEntity.Create(...);                           │   │   │
│  │     │  await repository.Add(order);                                          │   │   │
│  │     │  ChangeTracker: [Basket: Unchanged, Order: Added]                      │   │   │
│  │     └────────────────────────────────────────────────────────────────────────┘   │   │
│  │                                        ↓                                          │   │
│  │  3. Update Basket (Modified)                                                     │   │
│  │     ┌────────────────────────────────────────────────────────────────────────┐   │   │
│  │     │  basket.Checkout(transactionId);                                       │   │   │
│  │     │  repository.Update(basket);                                            │   │   │
│  │     │  ChangeTracker: [Basket: Modified, Order: Added]                       │   │   │
│  │     └────────────────────────────────────────────────────────────────────────┘   │   │
│  │                                        ↓                                          │   │
│  │  4. Add OutboxMessage (Added)                                                    │   │
│  │     ┌────────────────────────────────────────────────────────────────────────┐   │   │
│  │     │  var outbox = OutboxMessageEntity.Create(...);                         │   │   │
│  │     │  await outboxRepo.AddMessageAsync(outbox);                             │   │   │
│  │     │  ChangeTracker: [Basket: Modified, Order: Added, Outbox: Added]        │   │   │
│  │     └────────────────────────────────────────────────────────────────────────┘   │   │
│  │                                        ↓                                          │   │
│  │  5. SaveChanges() - SQL Generation                                               │   │
│  │     ┌────────────────────────────────────────────────────────────────────────┐   │   │
│  │     │  await repository.SaveChangesAsync();                                  │   │   │
│  │     │                                                                        │   │   │
│  │     │  DetectChanges() → Tìm tất cả entities có thay đổi                     │   │   │
│  │     │       ↓                                                                  │   │   │
│  │     │  Generate SQL Commands:                                                 │   │   │
│  │     │  • UPDATE Baskets WHERE Id = ...                                        │   │   │
│  │     │  • INSERT INTO Orders VALUES (...)                                      │   │   │
│  │     │  • INSERT INTO OutboxMessages VALUES (...)                              │   │   │
│  │     │       ↓                                                                  │   │   │
│  │     │  ExecuteAll trong Transaction                                           │   │   │
│  │     │       ↓                                                                  │   │   │
│  │     │  AcceptAllChanges() → [All: Unchanged]                                  │   │   │
│  │     └────────────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                                  │   │
│  └──────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### SQL Generation Process

Quá trình tạo SQL commands là một trong những phức tạp nhất của EF Core. Hệ thống phải phân tích các thay đổi trong ChangeTracker, xác định các bảng và cột liên quan, tạo các SQL commands tối ưu, và thực thi chúng trong đúng thứ tự để đảm bảo referential integrity.

EF Core sử dụng `IRelationalCommandBuilder` để tạo các SQL commands. Mỗi entity type có một mapping configuration được xác định khi DbContext được khởi tạo, bao gồm table name, column names, data types, và các constraints. Dựa trên thông tin này và trạng thái của entity, EF Core tạo ra SQL commands phù hợp.

Đối với INSERT operations, EF Core tạo ra câu lệnh INSERT với tất cả các column values. Đối với UPDATE operations, nó chỉ tạo SET clause cho các columns đã thay đổi (partial update) để tối ưu performance. Đối với DELETE, nó tạo DELETE với WHERE clause dựa trên primary key.

---

## Domain Event Dispatch Timing

Một trong những khía cạnh quan trọng nhất của transaction flow là khi nào Domain Events được dispatch. Trong dự án này, Domain Events được dispatch CHỈ SAU KHI SQL được thực thi thành công và TRƯỚC KHI transaction được commit.

```csharp
// Microsoft.EntityFrameworkCore (Simplified)
public virtual async Task<int> SaveChangesAsync(
    bool acceptAllChangesOnSuccess = true,
    CancellationToken cancellationToken = new CancellationToken())
{
    // ... (SQL generation and execution)
    
    // === BƯỚC QUAN TRỌNG: Dispatch Domain Events ===
    // Dispatch SAU SQL execution nhưng TRƯỚC commit
    await DispatchDomainEvents();
    
    // ... (Accept changes)
    
    return entries.Length;
}
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SAVE CHANGES INTERNAL FLOW                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  dbContext.SaveChangesAsync()                                                │
│           │                                                                   │
│           ├──► 1. DetectChanges()                                            │
│           │         │                                                         │
│           │         ▼                                                         │
│           ├──► 2. Generate SQL                                               │
│           │         │                                                         │
│           │         ▼                                                         │
│           ├──► 3. EXECUTE SQL (trong transaction)                            │
│           │         │                                                         │
│           │         ├──► INSERT Orders                                        │
│           │         ├──► UPDATE Baskets                                       │
│           │         └──► INSERT OutboxMessages                                │
│           │         │                                                         │
│           │         ▼                                                         │
│           ├──► 4. SQL COMMIT (dữ liệu đã persisted)                          │
│           │         │                                                         │
│           │         ▼                                                         │
│           ├──► 5. DispatchDomainEvents() ◄── CHỈ SAU KHI COMMIT              │
│           │         │                                                         │
│           │         ├──► BasketCheckoutDomainEventHandler                    │
│           │         │         │                                               │
│           │         │         └──► Add message to Outbox (already tracked)    │
│           │         │                                                         │
│           └──► 6. AcceptAllChanges()                                         │
│                                                                              │
│  ⚠️  KEY POINT: Domain Events CHỈ được dispatch SAU KHI SQL COMMIT          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

Điều này có nghĩa là khi `BasketCheckoutDomainEventHandler` được gọi, OutboxMessage đã được INSERT vào database và sẵn sàng để được commit cùng với transaction. Đây là design cực kỳ quan trọng vì nó đảm bảo rằng domain event chỉ được "handled" (trong trường hợp này là push vào outbox) khi và chỉ khi business data đã được persisted thành công.

```csharp
// File: src/Services/Basket/Core/Basket.Application/Features/Basket/EventHandlers/Domain/BasketCheckoutDomainEventHandler.cs

/// <summary>
/// Handler cho BasketCheckoutDomainEvent
/// Thêm message vào Outbox table để publish lên message broker
/// </summary>
public sealed class BasketCheckoutDomainEventHandler(
    IOutboxRepository outboxRepo,
    ILogger<BasketCheckoutDomainEventHandler> logger) : INotificationHandler<BasketCheckoutDomainEvent>
{
    public async Task Handle(BasketCheckoutDomainEvent @event, CancellationToken cancellationToken)
    {
        logger.LogInformation("Domain Event handled: {DomainEvent}", @event.GetType().Name);
        
        await PushToOutboxAsync(@event, cancellationToken);
    }

    private async Task PushToOutboxAsync(BasketCheckoutDomainEvent @event, CancellationToken cancellationToken)
    {
        // Tạo Integration Event từ Domain Event
        var message = new BasketCheckoutIntegrationEvent
        {
            Id = Guid.NewGuid().ToString(),
            BasketId = @event.Basket.Id,
            Customer = new CustomerIntegrationEvent
            {
                Id = @event.Customer.Id,
                Name = @event.Customer.Name,
                Email = @event.Customer.Email,
                Phone = @event.Customer.Phone,
                Address = @event.Customer.Address
            },
            ShippingAddress = new AddressIntegrationEvent
            {
                Line1 = @event.ShippingAddress.Line1,
                Line2 = @event.ShippingAddress.Line2,
                City = @event.ShippingAddress.City,
                State = @event.ShippingAddress.State,
                PostalCode = @event.ShippingAddress.PostalCode,
                Country = @event.ShippingAddress.Country
            },
            Discount = new DiscountIntegrationEvent
            {
                Code = @event.Discount?.Code,
                Amount = @event.Discount?.Amount ?? 0
            },
            Items = @event.Basket.Items?.Select(item => new CartItemIntegrationEvent
            {
                ProductId = item.ProductId,
                ProductName = item.ProductName,
                UnitPrice = item.UnitPrice,
                Quantity = item.Quantity,
                FinalPrice = item.FinalPrice
            }).ToList() ?? new List<CartItemIntegrationEvent>(),
            OccurredOn = DateTimeOffset.UtcNow
        };

        // Tạo OutboxMessageEntity
        var outboxMessage = OutboxMessageEntity.Create(
            id: Guid.NewGuid(),
            eventType: message.EventType!,
            content: JsonConvert.SerializeObject(message),
            occurredOnUtc: DateTimeOffset.UtcNow);

        // Thêm vào Outbox
        var result = await outboxRepo.AddMessageAsync(outboxMessage, cancellationToken);
        
        if (!result)
        {
            logger.LogWarning("Failed to add message to outbox. BasketId: {BasketId}", @event.Basket.Id);
            throw new InvalidOperationException("Failed to add message to outbox");
        }

        logger.LogInformation("Successfully added basket checkout event to outbox...");
        
        // ⚠️ QUAN TRỌNG: Tại thời điểm này:
        // - Basket đã được UPDATE (trong transaction)
        // - Order đã được INSERT (trong transaction)
        // - OutboxMessage đã được ADDED to ChangeTracker (trong transaction)
        //
        // NHƯNG CHƯA COMMIT!
        // OutboxMessage sẽ được INSERT vào DB khi SaveChangesAsync() hoàn tất
        // Và transaction sẽ được commit sau đó
    }
}
```

---

## Tóm tắt các hàm chính

| Phase | Hàm | File | EF Core Method | Mô tả |
|-------|-----|------|-----------------|-------|
| **Open** | `OpenTransactionAsync()` | `BasketRepository.cs` | `dbContext.Database.BeginTransactionAsync()` | Mở transaction mới hoặc trả về transaction hiện tại |
| **Add Data** | `Add(entity)` | `BasketRepository.cs` | `dbContext.AddAsync()` | Thêm entity mới vào ChangeTracker với state = Added |
| **Add Data** | `Update(entity)` | `BasketRepository.cs` | `dbContext.Update()` | Cập nhật entity trong ChangeTracker với state = Modified |
| **Add Data** | `AddMessageAsync()` | `OutboxRepository.cs` | `dbContext.AddAsync()` | Thêm OutboxMessage vào ChangeTracker |
| **Save** | `SaveChangesAsync()` | `BasketRepository.cs` | `dbContext.SaveChangesAsync()` | Thực thi SQL commands và dispatch domain events |
| **Commit** | `CommitAsync()` | `BasketRepository.cs` | `dbContext.Database.CommitTransactionAsync()` | Commit transaction và persist tất cả changes |
| **Rollback** | `RollbackAsync()` | `BasketRepository.cs` | `dbContext.Database.RollbackTransactionAsync()` | Rollback transaction khi có lỗi |
| **Dispose** | `DisposeAsync()` | `BasketRepository.cs` | `transaction.DisposeAsync()` | Giải phóng transaction và connection resources |

---

## Câu hỏi thường gặp

### 1. Tại sao cần mở transaction một cách tường minh thay vì để EF Core tự động?

Trong dự án này, việc mở transaction tường minh cho phép kiểm soát chính xác khi nào transaction bắt đầu và kết thúc. Điều này đặc biệt quan trọng khi có nhiều thao tác database cần được thực hiện như một đơn vị atomic. Nếu để EF Core tự động quản lý, mỗi `SaveChanges()` sẽ mở và đóng transaction riêng, dẫn đến nhiều round-trips database và không đảm bảo tính nhất quán giữa các thao tác.

Ngoài ra, việc mở transaction tường minh cho phép sử dụng `CommitAsync()` và `RollbackAsync()` một cách rõ ràng trong code business logic, làm cho luồng xử lý dễ hiểu và debug hơn. Điều này cũng cho phép thực hiện các thao tác phức tạp như nested transactions hoặc savepoints nếu cần thiết.

### 2. Điều gì xảy ra nếu SaveChanges thất bại nhưng Commit đã được gọi?

Trong thực tế, nếu `SaveChangesAsync()` thất bại, một exception sẽ được throw và code sẽ nhảy đến catch block nơi `RollbackAsync()` được gọi. Tuy nhiên, nếu `SaveChangesAsync()` thành công nhưng `CommitAsync()` thất bại (ví dụ: do network issue khi gửi COMMIT command), transaction đã được "committed" ở mức EF Core nhưng có thể không được persist đúng cách ở mức database.

Để xử lý trường hợp này, code cần có retry logic hoặc sử dụng transaction with isolation level phù hợp. Trong hầu hết các trường hợp, nếu `SaveChangesAsync()` thành công, các SQL commands đã được execute thành công và chỉ cần COMMIT để hoàn tất. Việc COMMIT thất bại là hiếm và thường liên quan đến network issues hoặc database failures nghiêm trọng.

### 3. Tại sao Outbox message phải được thêm vào cùng một transaction?

Việc thêm Outbox message vào cùng transaction với business data là cốt lõi của Outbox pattern. Điều này đảm bảo rằng message chỉ được publish khi và chỉ khi business transaction thành công. Nếu không có cùng transaction, có thể xảy ra trường hợp business data được commit nhưng message không được thêm (hoặc ngược lại), dẫn đến inconsistency giữa services.

Outbox pattern giải quyết vấn đề "dual write" trong distributed systems. Thay vì cố gắng write data và publish message cùng lúc (có thể thất bại ở một trong hai operations), Outbox pattern lưu cả hai vào cùng database trong một transaction, sau đó một separate process đọc Outbox và publish messages một cách độc lập.

### 4. Domain Event và Integration Event khác nhau như thế nào?

Domain Event là events được raise trong phạm vi của một bounded context (service) và được xử lý synchronous (in-process) thông qua MediatR. Chúng được sử dụng để communicate giữa các aggregates trong cùng một aggregate root hoặc để trigger các side effects trong cùng service. Ví dụ: `BasketCheckoutDomainEvent` được raise khi basket được checkout và được xử lý bởi `BasketCheckoutDomainEventHandler` trong cùng Basket service.

Integration Event là events được publish ra ngoài service boundary, thông qua message broker như RabbitMQ. Chúng được sử dụng để communicate giữa các services khác nhau. Ví dụ: `BasketCheckoutIntegrationEvent` được publish lên RabbitMQ và được consume bởi các services khác như Order Service, Inventory Service để thực hiện các actions tương ứng.

### 5. Background worker publish Outbox messages như thế nào?

Background worker (OutboxBackgroundService) chạy định kỳ và thực hiện các bước sau: đầu tiên, nó query database để tìm các OutboxMessages có `ProcessedOnUtc IS NULL` (chưa được processed), sau đó serialize message content và publish lên RabbitMQ, tiếp theo nếu publish thành công, cập nhật `ProcessedOnUtc = Now()` và lưu vào database, cuối cùng nếu publish thất bại, tăng `AttemptCount` và ghi `LastErrorMessage`.

Quan trọng là message chỉ được publish SAU KHI transaction được commit, đảm bảo rằng message chỉ tồn tại khi và chỉ khi business data đã được persisted thành công. Background worker cũng có retry logic để xử lý temporary failures.

---

## Tài liệu tham khảo

Tài liệu này được tạo dựa trên phân tích code thực tế trong dự án ProgCoder Shop Microservices, bao gồm các files:

- `src/Services/Basket/Core/Basket.Application/Features/Basket/Commands/BasketCheckoutCommand.cs`
- `src/Services/Basket/Core/Basket.Infrastructure/Repositories/BasketRepository.cs`
- `src/Services/Basket/Core/Basket.Infrastructure/Repositories/OutboxRepository.cs`
- `src/Services/Basket/Core/Basket.Application/Features/Basket/EventHandlers/Domain/BasketCheckoutDomainEventHandler.cs`
- `src/Services/Basket/Core/Basket.Domain/Entities/ShoppingCartEntity.cs`
- `src/Services/Basket/Core/Basket.Domain/Entities/OutboxMessageEntity.cs`

Để biết thêm chi tiết về kiến trúc Event-Driven và Outbox pattern, vui lòng tham khảo tài liệu `DOMAIN_EVENT_ANALYSIS.md` trong cùng thư mục docs.