# Tài Liệu: ISender vs IMediator và Send vs Publish trong MediatR

**Phiên bản:** 1.1  
**Ngày cập nhật:** 06/02/2026  
**Tác giả:** AI Research Assistant  
**Nguồn:** Tavily, DeepWiki, Grep Search, Repomix (Codebase Analysis)

---

## Mục Lục

1. [Giới Thiệu](#1-giới-thiệu)
2. [ISender vs IMediator](#2-isender-vs-imediator)
   - 2.1 [Định Nghĩa](#21-định-nghĩa)
   - 2.2 [So Sánh Chi Tiết](#22-so-sánh-chi-tiết)
   - 2.3 [Khi Nào Dùng Cái Nào](#23-khi-nào-dùng-cái-nào)
3. [Send Request vs Publish Notification](#3-send-request-vs-publish-notification)
   - 3.1 [Bản Chất và Mục Đích](#31-bản-chất-và-mục-đích)
   - 3.2 [Ví Dụ Thực Tế](#32-ví-dụ-thực-tế)
   - 3.3 [So Sánh Tổng Quan](#33-so-sánh-tổng-quan)
4. [Dependency Injection](#4-dependency-injection)
5. [Case Study: ProgCoder Shop Microservices](#5-case-study-progcoder-shop-microservices)
   - 5.1 [Tổng Quan Phân Tích Codebase](#51-tổng-quan-phân-tích-codebase)
   - 5.2 [Cấu Trúc Interfaces Tùy Chỉnh](#52-cấu-trúc-interfaces-tùy-chỉnh)
   - 5.3 [Cách Sử Dụng ISender trong Dự Án](#53-cách-sử-dụng-isender-trong-dự-án)
   - 5.4 [Cách Sử Dụng IMediator trong Dự Án](#54-cách-sử-dụng-imediator-trong-dự-án)
   - 5.5 [Luồng Dữ Liệu Thực Tế](#55-luồng-dữ-liệu-thực-tế)
   - 5.6 [Thống Kê Sử Dụng trong Dự Án](#56-thống-kê-sử-dụng-trong-dự-án)
   - 5.7 [Best Practices từ Dự Án](#57-best-practices-từ-dự-án)
6. [Best Practices](#6-best-practices)
   - 6.1 [Interface Segregation](#61-interface-segregation)
   - 6.2 [Command vs Query Separation (CQRS)](#62-command-vs-query-separation-cqrs)
   - 6.3 [Notification Naming Convention](#63-notification-naming-convention)
   - 6.4 [Error Handling](#64-error-handling)
   - 6.5 [Tóm Tắt Quy Tắc Vàng](#65-tóm-tắt-quy-tắc-vàng)
7. [Kết Luận](#7-kết-luận)

---

## 1. Giới Thiệu

Tài liệu này cung cấp hướng dẫn toàn diện về sự khác biệt giữa `ISender` và `IMediator`, cũng như `Send Request` và `Publish Notification` trong thư viện **MediatR** cho .NET. Nội dung được tổng hợp từ nhiều nguồn uy tín bao gồm tài liệu chính thức, code repository, và các ví dụ thực tế từ cộng đồng.

### MediatR là gì?

MediatR là một thư viện .NET triển khai **Mediator Pattern**, giúp giảm coupling giữa các components trong ứng dụng bằng cách đóng gói cách thức giao tiếp giữa chúng.

---

## 2. ISender vs IMediator

### 2.1 Định Nghĩa

#### ISender

`ISender` là interface tập trung vào **gửi requests** và **tạo streams**. Đây là một phần của `IMediator`, cung cấp chức năng narrow và tập trung.

```csharp
public interface ISender
{
    Task<TResponse> Send<TResponse>(
        IRequest<TResponse> request, 
        CancellationToken cancellationToken = default);
    
    Task<object?> Send(
        object request, 
        CancellationToken cancellationToken = default);
    
    IAsyncEnumerable<TResponse> CreateStream<TResponse>(
        IStreamRequest<TResponse> request, 
        CancellationToken cancellationToken = default);
    
    IAsyncEnumerable<object?> CreateStream(
        object request, 
        CancellationToken cancellationToken = default);
}
```

#### IMediator

`IMediator` là interface chính, kế thừa từ cả `ISender` và `IPublisher`, cung cấp đầy đủ chức năng mediator.

```csharp
public interface IMediator : ISender, IPublisher
{
    // Kế thừa tất cả methods từ ISender và IPublisher
}

public interface IPublisher
{
    Task Publish(
        object notification, 
        CancellationToken cancellationToken = default);
    
    Task Publish<TNotification>(
        TNotification notification, 
        CancellationToken cancellationToken = default) 
        where TNotification : INotification;
}
```

### 2.2 So Sánh Chi Tiết

| Tiêu chí | ISender | IMediator |
|----------|---------|-----------|
| **Mục đích** | Chỉ tập trung vào sending requests và creating streams | Interface chính, cung cấp đầy đủ chức năng mediator |
| **Methods** | `Send()`, `CreateStream()` | `Send()`, `Publish()`, `CreateStream()` |
| **Notifications** | Không hỗ trợ | Hỗ trợ qua `Publish()` |
| **Scope** | Narrow - chỉ sending | Broad - sending + publishing |
| **Kế thừa** | Interface độc lập | Kế thừa từ `ISender` và `IPublisher` |

#### Mối Quan Hệ

```
IMediator
    ├── ISender (Send, CreateStream)
    └── IPublisher (Publish)
```

### 2.3 Khi Nào Dùng Cái Nào

#### Sử dụng ISender khi:

1. **Chỉ cần gửi commands/queries**, không cần publish notifications
2. **Vertical Slice Architecture** - mỗi slice chỉ cần khả năng send
3. **Giảm coupling** - service không cần biết đến notification system
4. **Unit testing đơn giản hơn** - chỉ mock một interface nhỏ
5. **Thay thế MediatR** - một số thư viện thay thế như SwitchMediator chỉ implement ISender

**Ví dụ:**

```csharp
public class CreateOrderHandler
{
    private readonly ISender _sender;
    
    public CreateOrderHandler(ISender sender)
    {
        _sender = sender;
    }
    
    public async Task<OrderResult> Handle(CreateOrderRequest request)
    {
        // Chỉ cần send command, không cần publish notification
        var result = await _sender.Send(new CreateOrderCommand(request));
        return result;
    }
}
```

#### Sử dụng IMediator khi:

1. **Cần cả send và publish** - khi component cần gửi commands và publish events
2. **Full mediator capabilities** - khi cần toàn bộ chức năng
3. **Legacy code** - code cũ đang dùng IMediator
4. **Simple scenarios** - không cần tách biệt concerns

**Ví dụ:**

```csharp
public class OrderService
{
    private readonly IMediator _mediator;
    
    public OrderService(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    public async Task<Guid> CreateOrder(CreateOrderRequest request)
    {
        // Send command
        var orderId = await _mediator.Send(new CreateOrderCommand(request));
        
        // Publish notification
        await _mediator.Publish(new OrderCreatedNotification(orderId));
        
        return orderId;
    }
}
```

---

## 3. Send Request vs Publish Notification

### 3.1 Bản Chất và Mục Đích

| Tiêu chí | Send Request | Publish Notification |
|----------|--------------|---------------------|
| **Pattern** | Command/Query Pattern | Event-Driven / Observer Pattern |
| **Mục đích** | Yêu cầu thực hiện một tác vụ cụ thể | Thông báo rằng một sự kiện đã xảy ra |
| **Kết quả** | Trả về kết quả (có thể có hoặc không) | Không trả về kết quả (fire-and-forget) |
| **Số handlers** | **Chỉ 1 handler** duy nhất | **Nhiều handlers** (0, 1, hoặc nhiều) |
| **Interface** | `IRequest<T>` / `IRequest` | `INotification` |
| **Handler** | `IRequestHandler<TRequest, TResponse>` | `INotificationHandler<TNotification>` |
| **Execution** | Đồng bộ, chờ kết quả | Bất đồng bộ, fire-and-forget |

#### Send Request

```csharp
// Command - thực hiện hành động, không trả về dữ liệu
public record CreateOrderCommand(CreateOrderDto Data) : IRequest<Guid>;

// Query - lấy dữ liệu, trả về kết quả
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto>;

// Handler - chỉ có 1 handler cho mỗi request
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repository;
    
    public CreateOrderCommandHandler(IOrderRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        // Xử lý logic
        var orderId = await _repository.CreateAsync(request.Data);
        return orderId; // Trả về kết quả
    }
}
```

**Đặc điểm:**
- Một request chỉ được xử lý bởi **duy nhất 1 handler**
- Có thể trả về kết quả (`IRequest<TResponse>`)
- Có thể không trả về (`IRequest` - void)
- Luôn đồng bộ (chờ handler hoàn thành)

#### Publish Notification

```csharp
// Notification - thông báo sự kiện đã xảy ra
public record OrderCreatedNotification(Guid OrderId, DateTime CreatedAt) : INotification;

// Handler 1 - Gửi email thông báo
public class SendEmailHandler : INotificationHandler<OrderCreatedNotification>
{
    private readonly IEmailService _emailService;
    
    public SendEmailHandler(IEmailService emailService)
    {
        _emailService = emailService;
    }
    
    public async Task Handle(OrderCreatedNotification notification, CancellationToken ct)
    {
        await _emailService.SendOrderConfirmation(notification.OrderId);
    }
}

// Handler 2 - Cập nhật inventory
public class UpdateInventoryHandler : INotificationHandler<OrderCreatedNotification>
{
    private readonly IInventoryService _inventoryService;
    
    public UpdateInventoryHandler(IInventoryService inventoryService)
    {
        _inventoryService = inventoryService;
    }
    
    public async Task Handle(OrderCreatedNotification notification, CancellationToken ct)
    {
        await _inventoryService.ReserveStock(notification.OrderId);
    }
}

// Handler 3 - Ghi log audit
public class AuditLogHandler : INotificationHandler<OrderCreatedNotification>
{
    private readonly IAuditService _auditService;
    
    public AuditLogHandler(IAuditService auditService)
    {
        _auditService = auditService;
    }
    
    public async Task Handle(OrderCreatedNotification notification, CancellationToken ct)
    {
        await _auditService.Log("OrderCreated", notification);
    }
}
```

**Đặc điểm:**
- Một notification có thể có **nhiều handlers** (0, 1, hoặc nhiều)
- Không trả về kết quả (`INotification`)
- Fire-and-forget pattern
- Các handlers chạy độc lập với nhau

### 3.2 Ví Dụ Thực Tế

#### Kịch bản: Tạo đơn hàng

```csharp
public class OrderService
{
    private readonly ISender _sender;
    private readonly IPublisher _publisher;

    public OrderService(ISender sender, IPublisher publisher)
    {
        _sender = sender;
        _publisher = publisher;
    }

    public async Task<Guid> CreateOrder(CreateOrderRequest request)
    {
        // 1. SEND - Thực hiện tạo đơn hàng (chỉ 1 handler)
        // Đây là tác vụ CHÍNH, có kết quả trả về
        var orderId = await _sender.Send(new CreateOrderCommand(request));
        
        // 2. PUBLISH - Thông báo đơn hàng đã được tạo (nhiều handlers)
        // Đây là các tác vụ PHỤ, không cần kết quả
        await _publisher.Publish(new OrderCreatedNotification(orderId, DateTime.UtcNow));
        
        return orderId;
    }
}
```

**Luồng xử lý:**

```
CreateOrderCommand (Send)
    │
    ▼
[CreateOrderHandler] ──► Tạo order trong DB ──► Trả về OrderId
    │
    ▼
OrderCreatedNotification (Publish)
    │
    ├──► [SendEmailHandler] ──► Gửi email xác nhận
    │
    ├──► [UpdateInventoryHandler] ──► Cập nhật kho
    │
    └──► [AuditLogHandler] ──► Ghi log
```

### 3.3 So Sánh Tổng Quan

| Aspect | Send Request | Publish Notification |
|--------|--------------|---------------------|
| **Tương tự** | Gọi hàm | Raise event |
| **Mô hình** | 1-to-1 | 1-to-many |
| **Mục đích** | Làm việc gì đó | Báo rằng việc gì đó đã xảy ra |
| **Kết quả** | Có | Không |
| **Ví dụ** | CreateOrder, GetUser, UpdateProfile | OrderCreated, UserRegistered, PaymentReceived |

#### Quy Tắc Quyết Định

**Dùng Send khi:**
- Cần thực hiện một tác vụ cụ thể
- Cần kết quả trả về
- Chỉ có một cách xử lý duy nhất
- Ví dụ: Tạo đơn hàng, cập nhật profile, lấy danh sách sản phẩm

**Dùng Publish khi:**
- Muốn thông báo một sự kiện đã xảy ra
- Có nhiều thành phần cần phản ứng với sự kiện
- Không cần kết quả trả về
- Ví dụ: Gửi email, cập nhật cache, ghi log, thông báo real-time

---

## 4. Dependency Injection

Theo code từ `ServiceRegistrar.cs` trong MediatR.Extensions.Microsoft.DependencyInjection:

```csharp
public static IServiceCollection AddMediatR(this IServiceCollection services, 
    params Assembly[] assemblies)
{
    // Register Mediator
    services.AddTransient<IMediator, Mediator>();
    
    // ISender và IPublisher là wrapper của cùng một IMediator instance
    services.AddTransient<ISender>(sp => sp.GetRequiredService<IMediator>());
    services.AddTransient<IPublisher>(sp => sp.GetRequiredService<IMediator>());
    
    // Register handlers
    RegisterHandlers(services, assemblies);
    
    return services;
}
```

**Quan trọng:** Cả `ISender` và `IPublisher` đều là wrapper của cùng một `IMediator` instance. Chúng được register như **transient services**.

### Ví dụ Registration trong Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// Đăng ký MediatR
builder.Services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssemblyContaining<Program>();
});

// Hoặc đăng ký với assembly cụ thể
builder.Services.AddMediatR(typeof(Program).Assembly);
```

---

## 5. Case Study: ProgCoder Shop Microservices

Phần này phân tích cách sử dụng MediatR trong dự án ProgCoder Shop Microservices dựa trên codebase thực tế.

### 5.1 Tổng Quan Phân Tích Codebase

| Thông số | Giá trị |
|----------|---------|
| **Số lượng ISender/IMediator usage** | ~94 matches |
| **Số lượng Handlers** | 100+ matches |
| **Custom interfaces** | `ICommandHandler`, `IQueryHandler` |

### 5.2 Cấu Trúc Interfaces Tùy Chỉnh

Dự án sử dụng các interfaces tùy chỉnh kế thừa từ MediatR để tách biệt rõ ràng giữa Command và Query:

```csharp
// File: BuildingBlocks/Contracts/CQRS/ICommandHandler.cs
public interface ICommandHandler<in TCommand, TResponse> : IRequestHandler<TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
}

public interface ICommandHandler<in TCommand> : IRequestHandler<TCommand, Unit>
    where TCommand : ICommand
{
}

// File: BuildingBlocks/Contracts/CQRS/IQueryHandler.cs
public interface IQueryHandler<in TQuery, TResponse> : IRequestHandler<TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
}
```

**Lợi ích:**
- Tách biệt rõ ràng giữa Command và Query
- Dễ dàng mở rộng thêm behaviors riêng của dự án
- Type safety tốt hơn

### 5.3 Cách Sử Dụng ISender trong Dự Án

#### API Endpoints - Chỉ inject ISender

```csharp
// File: Catalog.Api/Endpoints/Products/CreateProductEndpoint.cs
public class CreateProductEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/products", async (CreateProductDto dto, ISender sender) =>
        {
            var command = new CreateProductCommand(dto, Actor.System);
            var result = await sender.Send(command);
            return Results.Ok(result);
        });
    }
}
```

#### Chaining Commands với ISender

```csharp
// File: CreateProductCommand.cs
public class CreateProductCommandHandler(
    IMapper mapper,
    IDocumentSession session,
    IMinIOCloudService minIO,
    ISender sender) : ICommandHandler<CreateProductCommand, Guid>
{
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken cancellationToken)
    {
        // ... logic tạo product ...
        
        // Gọi nested command bằng ISender khi chỉ cần send
        if (entity.Published)
        {
            await sender.Send(new PublishProductCommand(entity.Id, command.Actor), cancellationToken);
        }

        return entity.Id;
    }
}
```

**Quy tắc:** Sử dụng `ISender` khi handler chỉ cần gửi commands khác mà không cần publish notifications.

### 5.4 Cách Sử Dụng IMediator trong Dự Án

#### Publish Domain Events

```csharp
// File: PublishProductCommand.cs
public class PublishProductCommandHandler(
    IDocumentSession session, 
    IMediator mediator) : ICommandHandler<PublishProductCommand, Guid>
{
    public async Task<Guid> Handle(PublishProductCommand command, CancellationToken cancellationToken)
    {
        await session.BeginTransactionAsync(cancellationToken);

        var entity = await session.LoadAsync<ProductEntity>(command.ProductId, cancellationToken)
                ?? throw new ClientValidationException(MessageCode.ProductIsNotExists, command.ProductId);

        entity.Publish(command.Actor.ToString());
        session.Store(entity);

        // Tạo domain event
        var @event = new UpsertedProductDomainEvent(
            entity.Id,
            entity.Name!,
            entity.Sku!,
            // ... other properties ...
        );

        // Publish notification bằng IMediator
        await mediator.Publish(@event, cancellationToken);
        await session.SaveChangesAsync(cancellationToken);

        return entity.Id;
    }
}
```

**Quy tắc:** Sử dụng `IMediator` khi handler cần publish domain events để thông báo cho các thành phần khác.

### 5.5 Luồng Dữ Liệu Thực Tế

```
CreateProductEndpoint (API)
    │
    ▼
ISender.Send(CreateProductCommand)
    │
    ▼
CreateProductCommandHandler
    │   ├── Tạo ProductEntity
    │   ├── Upload images
    │   └── Save to MartenDB
    │
    ├──► ISender.Send(PublishProductCommand) [Nếu Published = true]
    │       │
    │       ▼
    │   PublishProductCommandHandler
    │       │   ├── Load entity
    │       │   ├── entity.Publish()
    │       │   └── Save changes
    │       │
    │       └──► IMediator.Publish(UpsertedProductDomainEvent)
    │               │
    │               ├──► SyncProductToElasticsearchHandler
    │               ├──► InvalidateProductCacheHandler
    │               └──► PublishProductEventHandler
    │
    └──► Return ProductId
```

### 5.6 Thống Kê Sử Dụng trong Dự Án

| Pattern | Số lượng | Vị trí chính |
|---------|----------|--------------|
| `ISender` | ~60 | API Endpoints, Handlers (chỉ send) |
| `IMediator` | ~34 | Handlers (cần publish events) |
| `ICommandHandler` | ~50 | Command handlers |
| `IQueryHandler` | ~30 | Query handlers |
| `INotificationHandler` | ~20 | Domain event handlers |

### 5.7 Best Practices từ Dự Án

1. **Interface Segregation:**
   - API Endpoints chỉ inject `ISender`
   - Handlers chỉ cần send cũng dùng `ISender`
   - Handlers cần publish events mới dùng `IMediator`

2. **Naming Convention:**
   - Command: `[Action][Entity]Command` (ví dụ: `CreateProductCommand`)
   - Query: `Get[Entity]Query` (ví dụ: `GetProductQuery`)
   - Event: `[Entity][Action]DomainEvent` (ví dụ: `UpsertedProductDomainEvent`)

3. **Chaining Commands:**
   - Mỗi command có trách nhiệm đơn lẻ (SRP)
   - Sử dụng `ISender` để gọi nested commands
   - Domain events được publish sau khi persistence thành công

---

## 6. Best Practices

### 6.1 Interface Segregation

Tuân thủ **Interface Segregation Principle** - chỉ inject những gì cần thiết:

```csharp
// Tốt - Chỉ inject những gì cần
public class OrderQueryService
{
    private readonly ISender _sender;
    
    public OrderQueryService(ISender sender)
    {
        _sender = sender;
    }
}

// Tốt - Khi cần cả send và publish
public class OrderCommandService
{
    private readonly IMediator _mediator;
    
    public OrderCommandService(IMediator mediator)
    {
        _mediator = mediator;
    }
}

// Tốt - Chỉ cần publish
public class EventPublisher
{
    private readonly IPublisher _publisher;
    
    public EventPublisher(IPublisher publisher)
    {
        _publisher = publisher;
    }
}
```

### 6.2 Command vs Query Separation (CQRS)

```csharp
// Commands - thay đổi state
public record CreateOrderCommand(CreateOrderDto Data) : IRequest<Guid>;
public record UpdateOrderCommand(Guid OrderId, UpdateOrderDto Data) : IRequest;
public record DeleteOrderCommand(Guid OrderId) : IRequest;

// Queries - chỉ đọc dữ liệu
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto>;
public record GetOrdersQuery(int Page, int PageSize) : IRequest<PagedResult<OrderDto>>;
```

### 6.3 Notification Naming Convention

```csharp
// Tên notification nên ở dạng quá khứ - thể hiện sự kiện đã xảy ra
public record OrderCreatedNotification(Guid OrderId) : INotification;
public record UserRegisteredNotification(Guid UserId, string Email) : INotification;
public record PaymentReceivedNotification(Guid OrderId, decimal Amount) : INotification;

// Tránh dùng tên ở dạng mệnh lệnh
// ❌ Không nên: CreateOrderNotification
// ✅ Nên: OrderCreatedNotification
```

### 6.4 Error Handling

```csharp
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        try
        {
            // Xử lý logic
            var orderId = await CreateOrderAsync(request.Data);
            return orderId;
        }
        catch (Exception ex)
        {
            // Log lỗi
            _logger.LogError(ex, "Failed to create order");
            throw; // Re-throw để caller xử lý
        }
    }
}
```

### 6.5 Tóm Tắt Quy Tắc Vàng

| Quy tắc | Giải thích |
|---------|------------|
| **Send để làm việc** | Dùng Send khi cần thực hiện tác vụ và có thể cần kết quả |
| **Publish để thông báo** | Dùng Publish khi muốn thông báo sự kiện đã xảy ra |
| **ISender cho narrow scope** | Chỉ cần gửi requests, không cần notifications |
| **IMediator cho full scope** | Cần cả send và publish |
| **IPublisher cho events** | Chỉ cần publish notifications |

---

## 7. Kết Luận

### Tóm Tắt Quan Trọng

1. **ISender** là subset của **IMediator**, chỉ cung cấp chức năng send requests
2. **IMediator** kế thừa từ cả `ISender` và `IPublisher`, cung cấp đầy đủ chức năng
3. **Send** dùng cho 1-to-1 communication (Command/Query)
4. **Publish** dùng cho 1-to-many communication (Events)
5. Luôn inject interface nhỏ nhất có thể (Interface Segregation Principle)

### Bảng Quyết Định Nhanh

| Nhu cầu | Interface | Method | Pattern |
|---------|-----------|--------|---------|
| Gửi command/query, không cần notification | `ISender` | `Send()` | Command/Query |
| Publish event, không cần send | `IPublisher` | `Publish()` | Event-Driven |
| Cần cả send và publish | `IMediator` | `Send()`, `Publish()` | Mixed |
| 1 handler, cần kết quả | - | `Send()` | Request/Response |
| Nhiều handlers, không cần kết quả | - | `Publish()` | Pub/Sub |

### Tài Nguyên Tham Khảo

- **Repository:** [jbogard/MediatR](https://github.com/jbogard/MediatR)
- **Documentation:** MediatR GitHub Wiki
- **Package:** `MediatR` và `MediatR.Extensions.Microsoft.DependencyInjection`

---

## Lịch Sử Thay Đổi

| Phiên bản | Ngày | Mô tả |
|-----------|------|-------|
| 1.0 | 06/02/2026 | Phiên bản đầu tiên |
| 1.1 | 06/02/2026 | Thêm Case Study phân tích codebase ProgCoder Shop Microservices |

---

*Tài liệu được tạo bởi AI Research Assistant sử dụng công cụ Tavily, DeepWiki, Grep Search, và Repomix.*
