# Discriminated Union Pattern trong .NET

## A. Tổng quan về Discriminated Union Pattern

### 1. Định nghĩa

**Discriminated Union Pattern** (còn gọi là Tagged Union, Algebraic Data Type, hoặc Sum Type) là một design pattern cho phép định nghĩa một kiểu dữ liệu có thể chứa một trong nhiều giá trị có cấu trúc khác nhau. Mỗi giá trị có một "tag" (discriminator) để phân biệt chúng.

### 2. Các thành phần chính

| Thành phần | Mô tả |
|------------|-------|
| **Base Type** | Interface/class/abstract record cơ sở chung |
| **Discriminator** | Thuộc tính dùng để phân biệt các kiểu con (thường là enum hoặc string) |
| **Variants** | Các kiểu con cụ thể (sealed record/class) |

### 3. Lợi ích

- **Type Safety**: Compiler kiểm tra tất cả cases tại compile-time
- **Exhaustive Pattern Matching**: Đảm bảo xử lý tất cả trường hợp
- **Extensibility**: Dễ dàng thêm variant mới
- **Serialization**: Dễ dàng serialize/deserialize với discriminator

---

## B. Case Study từ Codebase ProgCoder Shop Microservices

### Case Study 1: Actor Value Object (Implementation hoàn chỉnh)

**File**: `src/Shared/Common/ValueObjects/Actor.cs`

Đây là một ví dụ điển hình về Discriminated Union Pattern trong dự án:

```csharp
// ============================================
// 1. Discriminator (Enum)
// ============================================
public enum ActorKind
{
    User,
    System,
    Job,
    Worker,
    Consumer
}

// ============================================
// 2. Base Type (Record Struct)
// ============================================
public readonly record struct Actor
{
    public ActorKind Kind { get; init; }  // Discriminator property
    public Guid? Id { get; init; }
    public string? Name { get; init; }

    // ============================================
    // 3. Factory Methods (Variants)
    // ============================================
    public static Actor User(Guid id, string name) => 
        new() { Kind = ActorKind.User, Id = id, Name = name };
    
    public static Actor System(string name) => 
        new() { Kind = ActorKind.System, Name = name };
    
    public static Actor Job(string name) => 
        new() { Kind = ActorKind.Job, Name = name };
    
    public static Actor Worker(string name) => 
        new() { Kind = ActorKind.Worker, Name = name };
    
    public static Actor Consumer(string name) => 
        new() { Kind = ActorKind.Consumer, Name = name };
}
```

**Sử dụng với Pattern Matching**:

```csharp
// Trong các command handlers
var actor = command.Actor;

// Pattern matching dựa trên Kind
string actorInfo = actor.Kind switch
{
    ActorKind.User => $"User: {actor.Name} (ID: {actor.Id})",
    ActorKind.System => $"System: {actor.Name}",
    ActorKind.Job => $"Background Job: {actor.Name}",
    ActorKind.Worker => $"Worker: {actor.Name}",
    ActorKind.Consumer => $"Message Consumer: {actor.Name}",
    _ => throw new ArgumentOutOfRangeException()
};
```

**Đặc điểm**:
- Sử dụng `record struct` cho lightweight value object
- Discriminator là enum `ActorKind`
- Factory methods để tạo các variants
- Dùng trong Audit logging và Authorization

---

### Case Study 2: Order Status với Switch Expression

**File**: `src/Services/Order/Core/Order.Domain/Enums/OrderStatus.cs`

```csharp
public enum OrderStatus
{
    Pending,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Canceled,
    Refunded
}
```

**Sử dụng trong Business Logic**:

**File**: `src/Services/Order/Core/Order.Application/Features/Order/Commands/UpdateOrderStatusCommand.cs`

```csharp
public async Task<Unit> Handle(UpdateOrderStatusCommand command, CancellationToken cancellationToken)
{
    var order = await unitOfWork.Orders.GetByIdAsync(command.OrderId, cancellationToken);
    if (order == null)
    {
        throw new NotFoundException(MessageCode.OrderNotFound);
    }

    // Pattern matching với switch expression
    switch (command.Status)
    {
        case OrderStatus.Canceled:
            order.Cancel(command.Reason);
            break;
        case OrderStatus.Refunded:
            order.Refund(command.Reason);
            break;
        case OrderStatus.Delivered:
            order.MarkAsDelivered();
            break;
        default:
            order.UpdateStatus(command.Status);
            break;
    }

    await unitOfWork.SaveChangesAsync(cancellationToken);
    return Unit.Value;
}
```

**State Transition với Discriminated Union**:

```csharp
// Order Entity - State pattern kết hợp với enum
public class OrderEntity
{
    public OrderStatus Status { get; private set; }

    public void TransitionTo(OrderStatus newStatus)
    {
        // Validate transition dựa trên current status
        var isValid = (Status, newStatus) switch
        {
            (OrderStatus.Pending, OrderStatus.Confirmed) => true,
            (OrderStatus.Pending, OrderStatus.Canceled) => true,
            (OrderStatus.Confirmed, OrderStatus.Processing) => true,
            (OrderStatus.Confirmed, OrderStatus.Canceled) => true,
            (OrderStatus.Processing, OrderStatus.Shipped) => true,
            (OrderStatus.Shipped, OrderStatus.Delivered) => true,
            (OrderStatus.Delivered, OrderStatus.Refunded) => true,
            _ => false
        };

        if (!isValid)
        {
            throw new DomainException(
                $"Cannot transition from {Status} to {newStatus}");
        }

        Status = newStatus;
    }
}
```

**Đặc điểm**:
- Enum là discriminator đơn giản
- Switch expression để xử lý từng trạng thái
- Tuple pattern cho state transition validation

---

### Case Study 3: Coupon Type với Conditional Logic

**File**: `src/Services/Discount/Core/Discount.Domain/Enums/CouponType.cs`

```csharp
public enum CouponType
{
    Fixed,      // Giảm giá cố định (ví dụ: 50,000 VND)
    Percentage  // Giảm giá theo phần trăm (ví dụ: 10%)
}
```

**Sử dụng trong Entity**:

**File**: `src/Services/Discount/Core/Discount.Domain/Entities/CouponEntity.cs`

```csharp
public class CouponEntity
{
    public CouponType Type { get; private set; }
    public decimal Value { get; private set; }
    public decimal? MaxDiscountAmount { get; private set; }  // Chỉ dùng cho Percentage

    public decimal CalculateDiscount(decimal originalAmount)
    {
        return Type switch
        {
            CouponType.Fixed => Math.Min(Value, originalAmount),
            CouponType.Percentage => 
            {
                var discount = originalAmount * (Value / 100);
                return MaxDiscountAmount.HasValue 
                    ? Math.Min(discount, MaxDiscountAmount.Value) 
                    : discount;
            },
            _ => throw new NotSupportedException($"Coupon type {Type} is not supported")
        };
    }

    public void Validate()
    {
        switch (Type)
        {
            case CouponType.Fixed:
                if (Value <= 0)
                    throw new DomainException("Fixed coupon value must be greater than 0");
                break;
            case CouponType.Percentage:
                if (Value is <= 0 or > 100)
                    throw new DomainException("Percentage must be between 0 and 100");
                break;
        }
    }
}
```

**Đặc điểm**:
- Mỗi variant có business logic khác nhau
- Property `MaxDiscountAmount` chỉ có ý nghĩa với `Percentage`
- Validation logic phụ thuộc vào discriminator

---

### Case Study 4: Database Provider Factory

**File**: `src/Services/Order/Worker/Order.Woker.Outbox/Factories/DatabaseProviderFactory.cs`

```csharp
public enum DatabaseType
{
    PostgreSql,
    MySql,
    SqlServer
}

public static class DatabaseProviderFactory
{
    public static DbContextOptionsBuilder UseDatabase(
        this DbContextOptionsBuilder options, 
        DatabaseType type, 
        string connectionString)
    {
        return type switch
        {
            DatabaseType.PostgreSql => options.UseNpgsql(connectionString),
            DatabaseType.MySql => options.UseMySql(connectionString, ServerVersion.AutoDetect(connectionString)),
            DatabaseType.SqlServer => options.UseSqlServer(connectionString),
            _ => throw new NotSupportedException($"Database type {type} is not supported")
        };
    }
}
```

**Đặc điểm**:
- Factory pattern kết hợp với discriminated union
- Mỗi variant khởi tạo một implementation khác nhau
- Dễ dàng thêm database mới (MongoDB, SQLite, etc.)

---

## C. Các Implementation Patterns trong .NET

### Pattern 1: Abstract Record với Sealed Records (C# 9+)

```csharp
// Base type
public abstract record PaymentMethod
{
    public abstract string Type { get; }
}

// Variants
public sealed record CreditCardPayment(
    string CardNumber,
    string ExpiryDate,
    string Cvv
) : PaymentMethod
{
    public override string Type => "CreditCard";
}

public sealed record PayPalPayment(
    string Email,
    string Token
) : PaymentMethod
{
    public override string Type => "PayPal";
}

public sealed record BankTransferPayment(
    string AccountNumber,
    string BankName
) : PaymentMethod
{
    public override string Type => "BankTransfer";
}

// Pattern Matching
public string ProcessPayment(PaymentMethod payment) => payment switch
{
    CreditCardPayment cc => $"Processing Credit Card: {cc.CardNumber[..4]}****",
    PayPalPayment pp => $"Processing PayPal: {pp.Email}",
    BankTransferPayment bt => $"Processing Bank Transfer: {bt.BankName}",
    _ => throw new NotSupportedException()
};
```

### Pattern 2: Interface với Discriminator Property

```csharp
// Interface với discriminator
public interface IPaymentMethod
{
    string Type { get; }
}

// Records implementing interface
public record CreditCardPayment(
    string CardNumber,
    string ExpiryDate
) : IPaymentMethod
{
    public string Type => "CreditCard";
}

public record PayPalPayment(
    string Email
) : IPaymentMethod
{
    public string Type => "PayPal";
}

// JSON Serialization với discriminator
public class PaymentMethodJsonConverter : JsonConverter<IPaymentMethod>
{
    public override IPaymentMethod? Read(ref Utf8JsonReader reader, 
        Type typeToConvert, JsonSerializerOptions options)
    {
        using var doc = JsonDocument.ParseValue(ref reader);
        var root = doc.RootElement;
        var type = root.GetProperty("type").GetString();
        
        return type switch
        {
            "CreditCard" => JsonSerializer.Deserialize<CreditCardPayment>(
                root.GetRawText(), options),
            "PayPal" => JsonSerializer.Deserialize<PayPalPayment>(
                root.GetRawText(), options),
            _ => throw new JsonException($"Unknown type: {type}")
        };
    }

    public override void Write(Utf8JsonWriter writer, 
        IPaymentMethod value, JsonSerializerOptions options)
    {
        JsonSerializer.Serialize(writer, value, value.GetType(), options);
    }
}
```

### Pattern 3: Enum-based Discriminated Union

```csharp
// Đơn giản, hiệu quả cho các trường hợp không cần data phức tạp
public enum NotificationChannel
{
    Email,
    Sms,
    Push,
    InApp
}

public record NotificationRequest(
    NotificationChannel Channel,
    string Recipient,
    string Content,
    Dictionary<string, object>? Metadata = null
);

public class NotificationService
{
    public Task SendAsync(NotificationRequest request)
    {
        return request.Channel switch
        {
            NotificationChannel.Email => SendEmailAsync(request),
            NotificationChannel.Sms => SendSmsAsync(request),
            NotificationChannel.Push => SendPushAsync(request),
            NotificationChannel.InApp => SendInAppAsync(request),
            _ => throw new NotSupportedException()
        };
    }
}
```

### Pattern 4: OneOf Library (NuGet)

```csharp
// Sử dụng thư viện OneOf cho type-safe discriminated unions
using OneOf;

// Định nghĩa return type có thể là Success hoặc Error
public OneOf<Success, ValidationError, NotFoundError> ProcessOrder(Guid orderId)
{
    var order = _repository.GetById(orderId);
    if (order == null)
        return new NotFoundError($"Order {orderId} not found");
    
    if (!order.IsValid())
        return new ValidationError("Order validation failed");
    
    order.Process();
    return new Success();
}

// Sử dụng
var result = ProcessOrder(orderId);

result.Switch(
    success => Console.WriteLine("Order processed successfully"),
    validationError => Console.WriteLine($"Validation failed: {validationError.Message}"),
    notFoundError => Console.WriteLine($"Not found: {notFoundError.Message}")
);

// Hoặc dùng Match
var message = result.Match(
    success => "Success",
    validationError => $"Validation Error: {validationError.Message}",
    notFoundError => $"Not Found: {notFoundError.Message}"
);
```

---

## D. Best Practices

### 1. Luôn sử dụng exhaustive pattern matching

```csharp
// SAI - Thiếu case
var result = status switch
{
    OrderStatus.Pending => "Pending",
    OrderStatus.Confirmed => "Confirmed"
    // Thiếu các case khác!
};

// ĐÚNG - Exhaustive matching
var result = status switch
{
    OrderStatus.Pending => "Pending",
    OrderStatus.Confirmed => "Confirmed",
    OrderStatus.Processing => "Processing",
    OrderStatus.Shipped => "Shipped",
    OrderStatus.Delivered => "Delivered",
    OrderStatus.Canceled => "Canceled",
    OrderStatus.Refunded => "Refunded",
    _ => throw new ArgumentOutOfRangeException(nameof(status))
};
```

### 2. Sử dụng `sealed` cho các variants

```csharp
// ĐÚNG - Ngăn chặn inheritance không kiểm soát
public sealed record CreditCardPayment : PaymentMethod;

// SAI - Có thể bị override
public record CreditCardPayment : PaymentMethod;
```

### 3. JSON Serialization

```csharp
// Luôn bao gồm discriminator trong JSON
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    Converters = { new JsonStringEnumConverter() }
};

// Output: { "type": "CreditCard", "cardNumber": "1234..." }
```

### 4. Validation tập trung

```csharp
public abstract record PaymentMethod
{
    public abstract string Type { get; }
    
    // Validation chung cho tất cả variants
    public virtual void Validate()
    {
        if (string.IsNullOrEmpty(Type))
            throw new ValidationException("Type is required");
    }
}

public sealed record CreditCardPayment : PaymentMethod
{
    public override void Validate()
    {
        base.Validate();
        if (CardNumber?.Length != 16)
            throw new ValidationException("Invalid card number");
    }
}
```

---

## E. Khi nào nên sử dụng Discriminated Union

| Trường hợp | Ví dụ |
|------------|-------|
| **State Machine** | OrderStatus, PaymentStatus |
| **Kết quả Operation** | Success/Error variants |
| **Event Types** | Domain events, Integration events |
| **Configuration** | Database providers, Cache strategies |
| **UI Components** | Button variants, Alert types |

---

## F. Tài liệu tham khảo

- [Microsoft Docs - Pattern Matching](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)
- [Record Types in C#](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/records)
- [OneOf Library](https://github.com/mcintyre321/OneOf)
- [Functional Programming in C#](https://docs.microsoft.com/en-us/dotnet/csharp/functional/)

---

*Document được tạo từ phân tích codebase ProgCoder Shop Microservices*
