# Phân tích chi tiết về Domain Event trong ProgCoder Shop Microservices

## Mục lục

1. [Tổng quan về Domain Event](#1-tổng-quan-về-domain-event)
2. [Cơ chế hoạt động của Domain Event](#2-cơ-chế-hoạt-động-của-domain-event)
3. [Kiến trúc Event-Driven trong dự án](#3-kiến-trúc-event-driven-trong-dự-án)
4. [Vai trò của Interceptor trong việc dispatch events](#4-vai-trò-của-interceptor-trong-việc-dispatch-events)
5. [Quy trình xử lý End-to-End](#5-quy-trình-xử-lý-end-to-end)
6. [Ví dụ cụ thể: Order Creation Workflow](#6-ví-dụ-cụ-thể-order-creation-workflow)
7. [Tại sao Domain Event giúp Aggregate giao tiếp hiệu quả](#7-tại-sao-domain-event-giúp-aggregate-giao-tiếp-hiệu-quả)
8. [Phân biệt Domain Event vs Integration Event](#8-phân-biệt-domain-event-vs-integration-event)
9. [Kết luận](#9-kết-luận)

---

## 1. Tổng quan về Domain Event

### 1.1 Định nghĩa

Domain Event là một pattern trong Domain-Driven Design (DDD) đại diện cho một sự kiện quan trọng xảy ra trong domain mà các bên khác có thể quan tâm. Trong dự án ProgCoder Shop Microservices, Domain Event được sử dụng như cơ chế chính để các Aggregate giao tiếp với nhau mà không cần coupling trực tiếp.

### 1.2 Các Domain Event trong dự án

Dự án hiện có 17 Domain Events được định nghĩa:

**Order Service:**
- `OrderCreatedDomainEvent`
- `OrderCancelledDomainEvent`
- `OrderDeliveredDomainEvent`

**Basket Service:**
- `BasketCheckoutDomainEvent`
- `CustomerDomainEvent`
- `AddressDomainEvent`
- `DiscountDomainEvent`

**Catalog Service:**
- `DeletedUnPublishedProductDomainEvent`
- `UpsertedProductDomainEvent`

**Inventory Service:**
- `ReservationCreatedDomainEvent`
- `ReservationExpiredDomainEvent`
- `ReservationReleasedDomainEvent`
- `ReservationCommittedDomainEvent`
- `LocationChangedDomainEvent`
- `StockChangedDomainEvent`
- `TransferOutDomainEvent`
- `InventoryItemDeletedDomainEvent`

---

## 2. Cơ chế hoạt động của Domain Event

### 2.1 Cấu trúc Base Class

Mọi Aggregate Root đều kế thừa từ `Aggregate<TId>` - abstract class cung cấp cơ sở hạ tầng để quản lý domain events:

```csharp
public abstract class Aggregate<TId> : Entity<TId>, IAggregate<TId>
{
    private readonly List<IDomainEvent> _domainEvents = new();
    
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    public void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
    
    public IDomainEvent[] ClearDomainEvents()
    {
        IDomainEvent[] dequeuedEvents = _domainEvents.ToArray();
        _domainEvents.Clear();
        return dequeuedEvents;
    }
}
```

### 2.2 Cách Aggregate phát ra Event

Khi trạng thái của Aggregate thay đổi, nó tự động thêm Domain Event vào danh sách nội bộ:

```csharp
public sealed record OrderCreatedDomainEvent(OrderEntity Order) : IDomainEvent;
```

Aggregate sẽ raise event khi có thay đổi quan trọng:

```csharp
public sealed class OrderEntity : Aggregate<Guid>
{
    public void OrderCreated()
    {
        Status = OrderStatus.Created;
        AddDomainEvent(new OrderCreatedDomainEvent(this));
    }
}
```

---

## 3. Kiến trúc Event-Driven trong dự án

### 3.1 Tổng quan kiến trúc

Dự án sử dụng kiến trúc Event-Driven kết hợp với CQRS pattern. Events được phân loại thành hai loại chính:

| Loại Sự Kiện | Phạm vi | Môi trường | Cơ chế giao tiếp |
|--------------|---------|------------|------------------|
| **Domain Event** | Trong một Service (Bounded Context) | In-Process | MediatR (Bus trong bộ nhớ) |
| **Integration Event** | Giữa các Services | Out-of-Process | RabbitMQ (Message Broker) |

### 3.2 Sơ đồ kiến trúc

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DOMAIN EVENT ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐         ┌──────────────────┐                     │
│   │   Aggregate  │         │   MediatR Bus    │                     │
│   │   (Order)    │──Event──▶│   (In-Process)   │───Handler──────────▶│
│   │              │         │                  │                     │
│   └──────────────┘         └──────────────────┘                     │
│         │                                                  │         │
│         │ DispatchDomainEvents                            │         │
│         │ Interceptor                                     │         │
│         ▼                                                  ▼         │
│   ┌────────────────────────────────────────────────────────────┐    │
│   │                    EF Core Change Tracker                  │    │
│   └────────────────────────────────────────────────────────────┘    │
│         │                                                  │         │
│         │ SaveChangesAsync()                               │         │
│         ▼                                                  ▼         │
│   ┌──────────────┐         ┌──────────────────┐                     │
│   │  Database    │         │   Outbox Table   │──Background──▶Rabbit│
│   │  (SQL Server)│         │                  │      Worker  │  MQ   │
│   └──────────────┘         └──────────────────┘                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Vai trò của Interceptor trong việc dispatch events

### 4.1 Vấn đề cần giải quyết

Tại sao không publish event trực tiếp từ Aggregate? Vì:
1. Transaction có thể fail sau khi event được publish
2. Dữ liệu và event cần được commit đồng thời
3. Nếu database transaction fail, event không nên được publish

### 4.2 Giải pháp: DispatchDomainEventsInterceptor

```csharp
public class DispatchDomainEventsInterceptor(IMediator mediator) : SaveChangesInterceptor
{
    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, 
        InterceptionResult<int> result, 
        CancellationToken cancellationToken = default)
    {
        await DispatchDomainEvents(eventData.Context);
        return await base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    public async Task DispatchDomainEvents(DbContext? context)
    {
        if (context == null) return;

        var aggregates = context.ChangeTracker
            .Entries<IAggregate>()
            .Where(a => a.Entity.DomainEvents.Any())
            .Select(a => a.Entity);

        var domainEvents = aggregates
            .SelectMany(a => a.DomainEvents)
            .ToList();

        aggregates.ToList().ForEach(a => a.ClearDomainEvents());

        foreach (var domainEvent in domainEvents)
            await mediator.Publish(domainEvent);
    }
}
```

### 4.3 Quy trình xử lý của Interceptor

1. **Chặn SaveChanges**: Interceptor tự động được gọi trước khi dữ liệu được persist
2. **Tìm các Aggregate có Events**: Scan ChangeTracker để tìm entities implement IAggregate
3. **Extract Events**: Trích xuất tất cả events từ các aggregates
4. **Clear Events**: Xóa events khỏi aggregate để tránh duplicate processing
5. **Publish qua MediatR**: Gửi events đến các handlers tương ứng

---

## 5. Quy trình xử lý End-to-End

### 5.1 Luồng xử lý chi tiết

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        EVENT PROCESSING WORKFLOW                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. COMMAND LAYER                                                            │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  CreateOrderCommandHandler                                      │      │
│     │  ├── Validate input                                             │      │
│     │  ├── Create OrderEntity                                         │      │
│     │  ├── Call order.OrderCreated()                                  │      │
│     │  └── await unitOfWork.SaveChangesAsync()                        │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│  2. DOMAIN LAYER                                                              │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  OrderEntity                                                    │      │
│     │  ├── Update Status = Created                                    │      │
│     │  └── AddDomainEvent(new OrderCreatedDomainEvent(this))          │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│  3. INFRASTRUCTURE LAYER (Interceptor)                                       │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  DispatchDomainEventsInterceptor                                │      │
│     │  ├── Detect ChangeTracker.Entries with DomainEvents             │      │
│     │  ├── Extract events from aggregates                             │      │
│     │  ├── Clear events from aggregates                               │      │
│     │  └── mediator.Publish(domainEvent) via MediatR                  │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│  4. EVENT HANDLERS                                                           │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  OrderCreatedDomainEventHandler                                 │      │
│     │  ├── Log event                                                  │      │
│     │  └── PushToOutboxAsync(@event)                                  │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│  5. DATABASE TRANSACTION                                                     │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  Commit:                                                       │      │
│     │  ├── Order row inserted                                         │      │
│     │  └── OutboxMessage row inserted (status: Pending)              │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│  6. BACKGROUND WORKER                                                        │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  OutboxProcessor                                                │      │
│     │  ├── Query OutboxMessages WHERE status = Pending                │      │
│     │  ├── Serialize and publish to RabbitMQ                         │      │
│     │  └── Update status = Published                                  │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│  7. OTHER SERVICES (Optional)                                                │
│     ┌─────────────────────────────────────────────────────────────────┐      │
│     │  Inventory Service (Consumer)                                   │      │
│     │  ├── Receive OrderCreatedIntegrationEvent                       │      │
│     │  └── Reserve inventory for order                                │      │
│     └─────────────────────────────────────────────────────────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Ví dụ cụ thể: Order Creation Workflow

### 6.1 Domain Events trong Order Service

**OrderCreatedDomainEvent.cs:**
```csharp
public sealed record OrderCreatedDomainEvent(OrderEntity Order) : IDomainEvent;
```

**OrderCancelledDomainEvent.cs:**
```csharp
public sealed record OrderCancelledDomainEvent(OrderEntity Order) : IDomainEvent;
```

**OrderDeliveredDomainEvent.cs:**
```csharp
public sealed record OrderDeliveredDomainEvent(OrderEntity Order) : IDomainEvent;
```

### 6.2 Basket Service: BasketCheckoutDomainEvent

Đây là ví dụ phức tạp hơn về event chứa nhiều thông tin:

```csharp
public record BasketCheckoutDomainEvent(
    ShoppingCartEntity Basket,
    CustomerDomainEvent Customer,
    AddressDomainEvent ShippingAddress,
    DiscountDomainEvent Discount) : INotification;
```

### 6.3 Inventory Service: ReservationCreatedDomainEvent

```csharp
public sealed record ReservationCreatedDomainEvent(
    Guid ReservationId,
    Guid ProductId,
    string ProductName,
    Guid ReferenceId,
    Guid LocationId,
    long Quantity,
    DateTimeOffset? ExpiresAt) : IDomainEvent;
```

### 6.4 Ví dụ Code: Command Handler

```csharp
public sealed class CreateOrderCommandHandler(
    IUnitOfWork unitOfWork,
    IMapper mapper) : ICommandHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = new OrderEntity
        {
            Id = Guid.NewGuid(),
            CustomerId = command.CustomerId,
            Status = OrderStatus.Pending,
            CreatedAt = DateTimeOffset.UtcNow
        };

        // This triggers the domain event
        order.OrderCreated();

        await unitOfWork.Orders.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);

        return order.Id;
    }
}
```

---

## 7. Tại sao Domain Event giúp Aggregate giao tiếp hiệu quả

### 7.1 Loose Coupling (Giảm coupling)

**Không có Domain Event:**
```
OrderAggregate ──▶ InventoryService (direct dependency)
                 ──▶ NotificationService (direct dependency)
                 ──▶ PaymentService (direct dependency)
```

**Với Domain Event:**
```
OrderAggregate ──▶ OrderCreatedEvent
                         │
                         ├──▶ InventoryHandler
                         ├──▶ NotificationHandler
                         └──▶ PaymentHandler
```

### 7.2 Single Responsibility Principle

Mỗi Aggregate chỉ quan tâm đến business logic của chính nó:

```csharp
// OrderAggregate chỉ chịu trách nhiệm về order logic
public sealed class OrderEntity : Aggregate<Guid>
{
    public void OrderCreated()
    {
        Status = OrderStatus.Created;
        AddDomainEvent(new OrderCreatedDomainEvent(this));
    }
}

// Không cần biết về inventory, notification, hay payment
```

### 7.3 Extensibility (Dễ mở rộng)

Thêm handler mới không cần thay đổi Aggregate:

```csharp
// Thêm new behavior mà không touch OrderEntity
public sealed class OrderCreatedAnalyticsHandler 
    : INotificationHandler<OrderCreatedDomainEvent>
{
    public async Task Handle(OrderCreatedDomainEvent @event, CancellationToken ct)
    {
        // Send to analytics service
        await analyticsService.TrackOrderCreated(@event.Order.Id);
    }
}
```

### 7.4 Auditability (Khả năng kiểm tra)

Events lưu trữ lịch sử thay đổi:

```csharp
public record OrderCreatedDomainEvent(
    OrderEntity Order,
    DateTimeOffset CreatedAt = default,
    Guid? CorrelationId = null) : IDomainEvent;
```

### 7.5 Reliability (Độ tin cậy)

Outbox pattern đảm bảo event được deliver:

```
┌──────────────────────────────────────────────────────────────┐
│                    RELIABILITY GUARANTEE                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │ Aggregate│───▶│ Interceptor│───▶│  Outbox │───▶│ Rabbit │   │
│  │ raises  │    │  commits  │    │  Table  │    │   MQ    │   │
│  │  Event   │    │  TX + Event│    │(pending)│    │ Consumer│   │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘   │
│                                                               │
│  ✅ Atomic: Event chỉ được publish khi transaction thành công│
│  ✅ Idempotent: Có thể retry mà không duplicate               │
│  ✅ Ordered: Events được xử lý theo thứ tự                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Phân biệt Domain Event vs Integration Event

### 8.1 Domain Event

| Thuộc tính | Mô tả |
|------------|-------|
| **Phạm vi** | Trong một Bounded Context (Service) |
| **Giao tiếp** | In-process (MediatR) |
| **Purpose** | Thay đổi internal state, side effects trong cùng service |
| **Delivery** | Đồng bộ, trong cùng transaction |
| **Example** | `OrderCreatedDomainEvent` → cập nhật Read Model |

### 8.2 Integration Event

| Thuộc tính | Mô tả |
|------------|-------|
| **Phạm vi** | Giữa các Bounded Contexts (Services) |
| **Giao tiếp** | Out-of-process (RabbitMQ) |
| **Purpose** | Thông báo cho service khác |
| **Delivery** | Bất đồng bộ, eventual consistency |
| **Example** | `OrderCreatedIntegrationEvent` → Inventory Service giảm stock |

### 8.3 Ví dụ Integration Event

```csharp
public record OrderCreatedIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    List<OrderItemDto> Items,
    decimal TotalAmount) : IIntegrationEvent;
```

---

## 9. Kết luận

### 9.1 Lợi ích chính của Domain Event trong dự án

1. **Loose Coupling**: Aggregate không cần biết về handlers, chỉ cần raise events
2. **Single Responsibility**: Mỗi aggregate tập trung vào business logic của mình
3. **Extensibility**: Dễ dàng thêm behavior mới thông qua handlers
4. **Reliability**: Kết hợp Outbox pattern đảm bảo event delivery
5. **Auditability**: Events lưu trữ lịch sử thay đổi trong hệ thống
6. **Testability**: Dễ dàng test aggregate logic riêng biệt với handlers

### 9.2 Best Practices được áp dụng

- ✅ Sử dụng DispatchDomainEventsInterceptor để đảm bảo atomicity
- ✅ Kết hợp với Outbox pattern cho cross-service communication
- ✅ Sử dụng MediatR để decouple event publishing từ handling
- ✅ Phân biệt rõ ràng giữa Domain Events và Integration Events
- ✅ Mỗi event nên immutable và chỉ chứa data cần thiết

### 9.3 Khi nào nên sử dụng Domain Event

- Khi một thay đổi trong Aggregate cần trigger nhiều side effects
- Khi muốn giảm coupling giữa các phần của hệ thống
- Khi cần audit trail cho business events
- Khi muốn ability để replay events (event sourcing)

### 9.4 Khi nào KHÔNG nên sử dụng

- Khi side effect cần synchronous result từ handler
- Khi chỉ cần thay đổi đơn giản không có side effects
- Khi performance là critical và event overhead không chấp nhận được

---

## References

- [1] CQRS_ANALYSIS.md
- [2] DDD_ANALYSIS_FOR_AIAgent.md
- [3] CLEAN_ARCHITECTURE_ANALYSIS.md
- [4] Aggregate-Root-Value-Object-Documentation.md
- [5] EVENT_ARCHITECTURE_ANALYSIS.md
- [6] order-creation-workflow.md

---

**Tài liệu được tạo ngày:** 15/02/2026  
**Nguồn:** Phân tích từ code-graph-rag và lightrag-mcg