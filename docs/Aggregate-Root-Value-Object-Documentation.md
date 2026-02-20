# Tài liệu Aggregate Root và Value Object trong ProgCoder Shop Microservices

## Mục lục

- [1. Aggregate Root](#1-aggregate-root)
  - [1.1. Định nghĩa và Vai trò](#11-định-nghĩa-và-vai-trò)
  - [1.2. Cấu trúc Base Class](#12-cấu-trúc-base-class)
  - [1.3. Các Aggregate Root trong Dự án](#13-các-aggregate-root-trong-dự-án)
    - [1.3.1. OrderEntity (Order Service)](#131-orderentity-order-service)
    - [1.3.2. ShoppingCartEntity (Basket Service)](#132-shoppingcartentity-basket-service)
    - [1.3.3. ProductEntity (Catalog Service)](#133-productentity-catalog-service)
    - [1.3.4. CouponEntity (Discount Service)](#134-couponentity-discount-service)
- [2. Value Object](#2-value-object)
  - [2.1. Định nghĩa và Đặc điểm](#21-định-nghĩa-và-đặc-điểm)
  - [2.2. Các Triển khai Value Object trong Dự án](#22-các-triển-khai-value-object-trong-dự-án)
    - [2.2.1. Money (Record - Immutable)](#221-money-record---immutable)
    - [2.2.2. Actor (Record Struct - Lightweight)](#222-actor-record-struct---lightweight)
    - [2.2.3. Address (Class với Private Constructor)](#223-address-class-với-private-constructor)
    - [2.2.4. OrderNo (Class với Auto-Generation)](#224-orderno-class-với-auto-generation)
    - [2.2.5. Customer (Class với Protected Constructor)](#225-customer-class-với-protected-constructor)
    - [2.2.6. Discount (Class đơn giản)](#226-discount-class-đơn-giản)
    - [2.2.7. Product (Class cho Order Context)](#227-product-class-cho-order-context)
- [3. So sánh Các Pattern Triển khai](#3-so-sánh-các-pattern-triển-khai)
  - [3.1. Bảng So sánh Value Object](#31-bảng-so-sánh-value-object)
  - [3.2. Khi nào dùng Record vs Class?](#32-khi-nào-dùng-record-vs-class)
  - [3.3. Khi nào dùng `readonly record struct`, `record`, `class`?](#33-khi-nào-dùng-readonly-record-struct-record-class)
    - [3.3.1. `readonly record struct` - Khi nào dùng?](#331-readonly-record-struct---khi-nào-dùng)
    - [3.3.2. `record` (class record) - Khi nào dùng?](#332-record-class-record---khi-nào-dùng)
    - [3.3.3. `class` - Khi nào dùng?](#333-class---khi-nào-dùng)
  - [3.4. Bảng Quyết định](#34-bảng-quyết-định)
  - [3.5. Recommendations từ Dự án](#35-recommendations-từ-dự-án)
- [4. Best Practices trong Dự án](#4-best-practices-trong-dự-án)
  - [4.1. Factory Methods Pattern](#41-factory-methods-pattern)
  - [4.2. Domain Events Pattern](#42-domain-events-pattern)
  - [4.3. Encapsulation Pattern](#43-encapsulation-pattern)
  - [4.4. Audit Trail Pattern](#44-audit-trail-pattern)
- [5. Entity vs Value Object](#5-entity-vs-value-object)
  - [5.1. Phân biệt trong Dự án](#51-phân-biệt-trong-dự-án)
  - [5.2. OrderItemEntity - Entity hay Value Object?](#52-orderitementity---entity-hay-value-object)
- [6. Recommendations](#6-recommendations)
  - [6.1. Cải tiến có thể áp dụng](#61-cải-tiến-có-thể-áp-dụng)
  - [6.2. Anti-patterns cần tránh](#62-anti-patterns-cần-tránh)
- [7. Kết luận](#7-kết-luận)
- [Tài liệu Tham khảo](#tài-liệu-tham-khảo)

## Tổng quan

Tài liệu này mô tả chi tiết cách triển khai Aggregate Root và Value Object trong dự án ProgCoder Shop Microservices, dựa trên phân tích codebase và best practices hiện đại.

---

## 1. Aggregate Root

### 1.1. Định nghĩa và Vai trò

**Aggregate Root** là một cluster của các objects liên quan được xem như một đơn vị duy nhất cho mục đích thay đổi dữ liệu. Mỗi Aggregate có:
- Một **Root Entity** (Aggregate Root)
- **Boundary** xác định những gì bên trong và bên ngoài Aggregate
- **Invariants** (quy tắc nghiệp vụ) luôn được duy trì

### 1.2. Cấu trúc Base Class

```csharp
// File: src/Services/Order/Core/Order.Domain/Abstractions/Aggregate.cs
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

**Đặc điểm:**
- Kế thừa từ `Entity<TId>` (có Identity)
- Quản lý **Domain Events** để giao tiếp giữa các Bounded Context
- Generic `TId` cho phép linh hoạt kiểu ID (Guid, int, string, v.v.)

### 1.3. Các Aggregate Root trong Dự án

#### 1.3.1. OrderEntity (Order Service)

```csharp
// File: src/Services/Order/Core/Order.Domain/Entities/OrderEntity.cs
public sealed class OrderEntity : Aggregate<Guid>
{
    private readonly List<OrderItemEntity> _orderItems = new();

    public IReadOnlyList<OrderItemEntity> OrderItems => _orderItems.AsReadOnly();
    public Customer Customer { get; set; } = default!;
    public OrderNo OrderNo { get; set; } = default!;
    public Address ShippingAddress { get; set; } = default!;
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public Discount Discount { get; set; } = default!;
    
    // Calculated Properties
    public decimal TotalPrice => OrderItems.Sum(x => x.LineTotal);
    public decimal FinalPrice 
    { 
        get 
        { 
            var discountAmount = Discount?.DiscountAmount ?? 0m;
            return Math.Max(0, OrderItems.Sum(x => x.LineTotal) - discountAmount);
        } 
    }

    // Factory Method
    public static OrderEntity Create(Guid id, Customer customer, OrderNo orderNo, 
        Address shippingAddress, string? notes, string performedBy)
    {
        var order = new OrderEntity
        {
            Id = id,
            Customer = customer,
            OrderNo = orderNo,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Pending,
            Notes = notes,
            CreatedBy = performedBy,
            LastModifiedBy = performedBy
        };
        return order;
    }

    // Domain Methods
    public void AddOrderItem(Product product, int quantity)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);
        var orderItem = OrderItemEntity.Create(Guid.NewGuid(), Id, product, quantity, CreatedBy!);
        _orderItems.Add(orderItem);
    }

    public void CancelOrder(string reason, string performBy)
    {
        UpdateStatus(OrderStatus.Canceled, performBy);
        CancelReason = reason;
        AddDomainEvent(new OrderCancelledDomainEvent(this));
    }

    public void OrderDelivered(string performBy)
    {
        UpdateStatus(OrderStatus.Delivered, performBy);
        AddDomainEvent(new OrderDeliveredDomainEvent(this));
    }
}
```

**Phân tích:**
- **sealed class**: Ngăn chặn inheritance, bảo vệ business invariants
- **Private collection**: `_orderItems` chỉ expose qua `IReadOnlyList`
- **Factory Method**: `Create()` đảm bảo validation và khởi tạo đúng
- **Domain Methods**: `CancelOrder()`, `OrderDelivered()` encapsulate business logic và raise domain events
- **Calculated Properties**: `TotalPrice`, `FinalPrice` tính toán runtime

#### 1.3.2. ShoppingCartEntity (Basket Service)

```csharp
// File: src/Services/Basket/Core/Basket.Domain/Entities/ShoppingCartEntity.cs
[BsonCollection("Shopping_Carts")]
public sealed class ShoppingCartEntity : Aggregate<Guid>
{
    public string UserId { get; set; } = default!;
    public List<ShoppingCartItemEntity> Items { get; set; } = new();
    public decimal TotalPrice => Items.Sum(x => x.LineTotal);

    public static ShoppingCartEntity Create(string userId)
    {
        return new ShoppingCartEntity()
        {
            Id = Guid.NewGuid(),
            UserId = userId
        };
    }

    public ShoppingCartItemEntity AddOrIncreaseItem(Guid productId, string productName, 
        string productImage, decimal price, int quantity = 1)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);
        ArgumentOutOfRangeException.ThrowIfNegative(price);

        var existing = Find(productId);
        if (existing is not null)
        {
            existing.Increase(quantity);
            existing.UpdatePrice(price);
            return existing;
        }

        var item = ShoppingCartItemEntity.Create(productId, productName, productImage, price, quantity);
        Items.Add(item);
        return item;
    }
}
```

**Phân tích:**
- **MongoDB Attribute**: `[BsonCollection("Shopping_Carts")]` cho MongoDB mapping
- **Business Logic**: `AddOrIncreaseItem` tự động tăng số lượng nếu sản phẩm đã tồn tại
- **Validation**: Kiểm tra quantity và price ngay trong method

#### 1.3.3. ProductEntity (Catalog Service)

```csharp
// File: src/Services/Catalog/Core/Catalog.Domain/Entities/ProductEntity.cs
public sealed class ProductEntity : Aggregate<Guid>
{
    public string? Name { get; set; }
    public string? Sku { get; set; }
    public decimal Price { get; set; }
    public decimal? SalePrice { get; set; }
    public List<Guid>? CategoryIds { get; set; }
    public List<ProductImageEntity>? Images { get; set; }
    public ProductImageEntity? Thumbnail { get; set; }
    public bool Published { get; set; }
    public ProductStatus Status { get; set; }

    public static ProductEntity Create(Guid id, string name, string sku, ...)
    {
        return new ProductEntity
        {
            Id = id,
            Name = name,
            Sku = sku,
            Status = ProductStatus.OutOfStock,
            Published = false,
            // ...
        };
    }

    public void Publish(string performedBy)
    {
        Published = true;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
    }

    public void ChangeStatus(ProductStatus status, string performedBy)
    {
        if (Status == status)
            throw new DomainException(MessageCode.DecisionFlowIllegal);
        
        Status = status;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
    }
}
```

**Phân tích:**
- **Rich Model**: Nhiều properties và methods xử lý logic sản phẩm phức tạp
- **State Machine**: `ChangeStatus()` với validation chuyển trạng thái
- **Audit Trail**: Mọi thay đổi đều ghi lại `LastModifiedBy` và `LastModifiedOnUtc`

#### 1.3.4. CouponEntity (Discount Service)

```csharp
// File: src/Services/Discount/Core/Discount.Domain/Entities/CouponEntity.cs
public sealed class CouponEntity : Aggregate<Guid>
{
    public string Code { get; set; } = default!;
    public CouponType Type { get; set; }
    public double Value { get; set; }
    public int MaxUsage { get; set; }
    public int UsageCount { get; set; }
    public DateTime ValidFrom { get; set; }
    public DateTime ValidTo { get; set; }

    public static CouponEntity Create(Guid id, string code, string name, ...)
    {
        // Validation trong factory
        ArgumentNullException.ThrowIfNull(code);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(value);
        
        if (validFrom >= validTo)
            throw new ArgumentException("ValidFrom must be before ValidTo");

        if (type == CouponType.Percentage && value > 100)
            throw new ArgumentException("Percentage value cannot exceed 100");

        return new CouponEntity() { ... };
    }

    public void Apply()
    {
        if (!CanBeUsed())
            throw new InvalidOperationException("Coupon cannot be used");

        UsageCount++;
        if (UsageCount >= MaxUsage)
            Status = CouponStatus.OutOfStock;
    }

    public decimal CalculateDiscount(decimal originalAmount)
    {
        decimal discount = Type == CouponType.Fixed 
            ? (decimal)Value 
            : originalAmount * (decimal)(Value / 100.0);
            
        if (MaxDiscountAmount.HasValue && discount > MaxDiscountAmount.Value)
            discount = MaxDiscountAmount.Value;
            
        return discount > originalAmount ? originalAmount : discount;
    }

    public bool IsValid()
    {
        return Status == CouponStatus.Approved
            && DateTime.UtcNow >= ValidFrom
            && DateTime.UtcNow <= ValidTo
            && UsageCount < MaxUsage;
    }
}
```

**Phân tích:**
- **Complex Validation**: Nhiều business rules trong factory method
- **State Management**: `Apply()` tự động cập nhật usage count và status
- **Calculation Logic**: `CalculateDiscount()` xử lý cả fixed và percentage discount
- **Business Queries**: `IsValid()`, `CanBeUsed()`, `IsExpired()` cung cấp thông tin trạng thái

---

## 2. Value Object

### 2.1. Định nghĩa và Đặc điểm

**Value Object** là object được xác định bởi giá trị của nó, không phải identity. Đặc điểm:
- **Immutability**: Không thể thay đổi sau khi tạo
- **Value-based Equality**: So sánh bằng giá trị, không phải reference
- **No Identity**: Không có ID riêng
- **Replaceable**: Thay thế toàn bộ thay vì update

### 2.2. Các Triển khai Value Object trong Dự án

#### 2.2.1. Money (Record - Immutable)

```csharp
// File: src/Services/Catalog/Core/Catalog.Domain/ValueObjects/Money.cs
public sealed record Money(decimal Amount, string Currency)
{
    public static Money From(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException(MessageCode.MoneyCannotBeNegative);
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException(MessageCode.CurrencyIsRequired);
        return new Money(amount, currency.ToUpperInvariant());
    }
}
```

**Phân tích:**
- **`sealed record`**: Immutable by default, value-based equality tự động
- **Positional Parameters**: Ngắn gọn, rõ ràng
- **Factory Method**: `From()` với validation logic
- **Không có setter**: Tính bất biến được đảm bảo

#### 2.2.2. Actor (Record Struct - Lightweight)

```csharp
// File: src/Shared/Common/ValueObjects/Actor.cs
public enum ActorKind { User, System, Job, Worker, Consumer }

public readonly record struct Actor(ActorKind Kind, string Value)
{
    public static Actor User(string value) => new(ActorKind.User, value);
    public static Actor System(string value) => new(ActorKind.System, value);
    public static Actor Job(string value) => new(ActorKind.Job, value);
    public static Actor Worker(string value) => new(ActorKind.Worker, value);
    public static Actor Consumer(string value) => new(ActorKind.Consumer, value);

    public override string ToString()
    {
        if (Kind == ActorKind.User) return Value!;
        return $"{Kind.ToString().ToLowerInvariant()}:{Value}";
    }
}
```

**Phân tích:**
- **`readonly record struct`**: Stack allocation, rất lightweight
- **Discriminated Union Pattern**: Dùng enum + factory methods cho từng loại actor
- **Custom ToString():** Format đặc biệt cho non-user actors
- **Shared Value Object**: Dùng chung across services

#### 2.2.3. Address (Class với Private Constructor)

```csharp
// File: src/Services/Order/Core/Order.Domain/ValueObjects/Address.cs
public class Address
{
    public string AddressLine { get; set; } = default!;
    public string Subdivision { get; set; } = default!; // Ward / County
    public string City { get; set; } = default!;
    public string StateOrProvince { get; set; } = default!;
    public string Country { get; set; } = default!;
    public string PostalCode { get; set; } = default!;

    private Address(string addressLine, string subdivision, string city, 
        string country, string stateOrProvince, string postalCode)
    {
        AddressLine = addressLine;
        Subdivision = subdivision;
        City = city;
        Country = country;
        StateOrProvince = stateOrProvince;
        PostalCode = postalCode;
    }

    public static Address Of(string addressLine, string subdivision, string city, 
        string country, string stateOrProvince, string postalCode)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(addressLine);
        ArgumentException.ThrowIfNullOrWhiteSpace(subdivision);
        ArgumentException.ThrowIfNullOrWhiteSpace(city);
        ArgumentException.ThrowIfNullOrWhiteSpace(country);
        ArgumentException.ThrowIfNullOrWhiteSpace(stateOrProvince);
        ArgumentException.ThrowIfNullOrWhiteSpace(postalCode);

        return new Address(addressLine, subdivision, city, country, stateOrProvince, postalCode);
    }
}
```

**Phân tích:**
- **Class thay vì Record**: EF Core compatibility, nhiều properties phức tạp
- **Private Constructor**: Chỉ tạo được qua factory method
- **Validation đầy đủ**: Tất cả fields đều required
- **Mutable Properties**: Dùng setter để EF Core có thể hydrate (trade-off với immutability)

#### 2.2.4. OrderNo (Class với Auto-Generation)

```csharp
// File: src/Services/Order/Core/Order.Domain/ValueObjects/OrderNo.cs
public class OrderNo
{
    public string Value { get; }

    private OrderNo(string value) => Value = value;

    public static OrderNo Create()
    {
        var dateString = DateTimeOffset.Now.ToString("yyyyMMdd");
        var sequenceString = Guid.NewGuid().ToString().Split("-").First().ToUpper();
        var orderNumber = $"ORD-{dateString}-{sequenceString}";
        return new OrderNo(orderNumber);
    }

    public override string ToString() => Value;
}
```

**Phân tích:**
- **Auto-Generated Value**: Tự động tạo theo format `ORD-yyyyMMdd-GUID`
- **Immutable**: Chỉ có getter, không thể thay đổi sau tạo
- **Domain-Specific Format**: Format phù hợp với business requirements
- **Implicit Conversion**: `ToString()` cho phép dùng trực tiếp trong string interpolation

#### 2.2.5. Customer (Class với Protected Constructor)

```csharp
// File: src/Services/Order/Core/Order.Domain/ValueObjects/Customer.cs
public class Customer
{
    public Guid? Id { get; set; }
    public string PhoneNumber { get; set; } = default!;
    public string Name { get; set; } = default!;
    public string Email { get; set; } = default!;

    protected Customer() { }

    public static Customer Of(Guid? id, string phoneNumber, string name, string email)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(phoneNumber);
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentException.ThrowIfNullOrWhiteSpace(email);

        return new Customer
        {
            Id = id,
            PhoneNumber = phoneNumber,
            Name = name,
            Email = email
        };
    }
}
```

**Phân tích:**
- **Protected Constructor**: Cho phép EF Core và subclass (nếu cần) tạo instance
- **Optional Id**: `Guid?` cho phép guest checkout (chưa có account)
- **Contact Information**: Tập hợp thông tin liên hệ của customer
- **Validation cơ bản**: Kiểm tra null/whitespace

#### 2.2.6. Discount (Class đơn giản)

```csharp
// File: src/Services/Order/Core/Order.Domain/ValueObjects/Discount.cs
public class Discount
{
    public string CouponCode { get; set; }
    public decimal DiscountAmount { get; set; }

    private Discount(string couponCode, decimal discountAmount)
    {
        CouponCode = couponCode;
        DiscountAmount = discountAmount;
    }

    public static Discount Of(string couponCode, decimal discountAmount)
    {
        return new Discount(couponCode, discountAmount);
    }
}
```

**Phân tích:**
- **Snapshot Pattern**: Lưu trạng thái discount tại thời điểm áp dụng
- **Decoupling**: Không reference trực tiếp đến CouponEntity
- **Historical Accuracy**: Giữ giá trị discount ngay cả khi coupon thay đổi sau này

#### 2.2.7. Product (Class cho Order Context)

```csharp
// File: src/Services/Order/Core/Order.Domain/ValueObjects/Product.cs
public class Product
{
    public Guid Id { get; set; } = default!;
    public string Name { get; set; } = default!;
    public string ImageUrl { get; set; } = default!;
    public decimal Price { get; set; } = default!;

    private Product() { }

    public static Product Of(Guid id, string name, decimal price, string imageUrl)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        return new Product()
        {
            Id = id,
            Name = name,
            Price = price,
            ImageUrl = imageUrl
        };
    }
}
```

**Phân tích:**
- **Denormalization**: Lưu thông tin product cần thiết trong order
- **Snapshot Pattern**: Giữ giá trị tại thời điểm đặt hàng
- **Bounded Context Boundary**: Product VO này thuộc Order Context, khác với ProductEntity trong Catalog Context

---

## 3. So sánh Các Pattern Triển khai

### 3.1. Bảng So sánh Value Object

| Value Object | Type | Immutability | Use Case | Pros | Cons |
|-------------|------|--------------|----------|------|------|
| **Money** | `sealed record` | Full | Simple value with validation | Concise, value equality, immutable | Limited customization |
| **Actor** | `readonly record struct` | Full | Lightweight cross-cutting concern | Stack allocation, very fast | Limited to small data |
| **Address** | `class` | Partial (EF) | Complex object with many fields | EF Core compatible, flexible | Mutable properties |
| **OrderNo** | `class` | Full | Auto-generated identifier | Custom generation logic | More verbose |
| **Customer** | `class` | Partial (EF) | Entity-like value object | Protected ctor for inheritance | Mutable properties |
| **Discount** | `class` | Partial | Snapshot pattern | Simple, decoupled | Mutable properties |
| **Product** | `class` | Partial (EF) | Cross-context reference | Denormalized data | Data duplication |

### 3.2. Khi nào dùng Record vs Class?

#### Dùng `record` khi:
- Value object đơn giản (2-3 properties)
- Không cần EF Core mapping trực tiếp
- Cần immutability nghiêm ngặt
- Không cần custom constructor logic phức tạp

#### Dùng `class` khi:
- EF Core cần map trực tiếp (Owned Entity)
- Cần private/protected constructor
- Cần validation phức tạp trong factory
- Cần mutable properties (vì lý do technical nhưng vẫn giữ semantic immutability)

### 3.3. Khi nào dùng `readonly record struct`, `record`, `class`?

#### 3.3.1. `readonly record struct` - Khi nào dùng?

**Dùng khi:**
- **Value object đơn giản** (1-2 properties)
- **Cần performance cao** (stack allocation)
- **Không cần EF Core map trực tiếp**
- **Dùng chung across services** (cross-cutting concern)

**Ví dụ trong dự án - Actor:**

```csharp
public readonly record struct Actor(ActorKind Kind, string Value)
{
    public static Actor User(string value) => new(ActorKind.User, value);
    public static Actor System(string value) => new(ActorKind.System, value);
}
```

**Tại sao dùng `readonly record struct`:**
- **Nhẹ**: Stack allocation, không cần GC
- **Immutable**: Không thể thay đổi sau tạo
- **Value equality**: So sánh bằng giá trị tự động
- **Pass-by-value**: Copy toàn bộ khi truyền (phù hợp cho small data)

**Không dùng khi:**
- ❌ Nhiều hơn 3-4 properties (dễ gây stack overflow nếu nested)
- ❌ Cần EF Core lưu trữ trực tiếp
- ❌ Cần reference type behavior

#### 3.3.2. `record` (class record) - Khi nào dùng?

**Dùng khi:**
- **Value object đơn giản** (2-3 properties)
- **Cần immutability nghiêm ngặt**
- **Không cần EF Core map trực tiếp** (hoặc dùng như Owned Entity)
- **Validation đơn giản** trong factory method

**Ví dụ trong dự án - Money:**

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public static Money From(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException(MessageCode.MoneyCannotBeNegative);
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException(MessageCode.CurrencyIsRequired);
        return new Money(amount, currency.ToUpperInvariant());
    }
}
```

**Tại sao dùng `record`:**
- **Immutable by default**: Chỉ có `init` properties
- **Concise syntax**: `record Money(decimal Amount, string Currency)`
- **Value equality**: Tự động override `Equals` và `GetHashCode`
- **With-expressions**: `money with { Amount = 100 }`
- **Pattern matching**: Dễ dàng deconstruct

**So sánh với `readonly record struct`:**

| Đặc điểm | `record` (class) | `readonly record struct` |
|----------|-----------------|-------------------------|
| Allocation | Heap | Stack |
| Size limit | Không giới hạn | Nên < 64 bytes |
| Inheritance | Có thể sealed | Không thể |
| Boxing | Không | Có thể boxing |
| Use case | Complex VO | Simple, frequently copied VO |

#### 3.3.3. `class` - Khi nào dùng?

**Dùng khi:**
- **EF Core cần map trực tiếp** (Owned Entity)
- **Validation phức tạp** trong constructor
- **Nhiều properties** (> 3)
- **Cần private/protected constructor** cho EF Core
- **Mutable properties** vì lý do technical (nhưng vẫn giữ semantic immutability)

**Ví dụ 1 - Complex Value Object (Address):**

```csharp
public class Address
{
    public string AddressLine { get; set; } = default!;
    public string City { get; set; } = default!;
    public string Country { get; set; } = default!;
    // ... 6 properties total

    private Address(string addressLine, string city, string country)
    {
        // Private constructor cho EF Core
        AddressLine = addressLine;
        City = city;
        Country = country;
    }

    public static Address Of(string addressLine, string city, string country)
    {
        // Validation phức tạp
        ArgumentException.ThrowIfNullOrWhiteSpace(addressLine);
        // ...
        return new Address(addressLine, city, country);
    }
}
```

**Tại sao dùng `class`:**
- **6 properties**: Quá nhiều cho `record struct`
- **EF Core**: Cần lưu trong Order table (Owned Entity)
- **Complex validation**: Nhiều fields cần kiểm tra
- **Private constructor**: EF Core yêu cầu để materialize

**Ví dụ 2 - Auto-generated Value Object (OrderNo):**

```csharp
public class OrderNo
{
    public string Value { get; }

    private OrderNo(string value) => Value = value;

    public static OrderNo Create()
    {
        var dateString = DateTimeOffset.Now.ToString("yyyyMMdd");
        var sequenceString = Guid.NewGuid().ToString().Split("-").First().ToUpper();
        var orderNumber = $"ORD-{dateString}-{sequenceString}";
        return new OrderNo(orderNumber);
    }
}
```

**Tại sao dùng `class`:**
- **Custom generation logic**: Không thể dùng positional constructor
- **Immutable**: Chỉ có getter
- **Factory method**: `Create()` tự động generate

### 3.4. Bảng Quyết định

| Tiêu chí | `readonly record struct` | `record` | `class` |
|----------|------------------------|----------|---------|
| **Số properties** | 1-2 | 2-3 | 3+ |
| **EF Core map** | ❌ | ⚠️ (Owned) | ✅ |
| **Validation** | Đơn giản | Đơn giản | Phức tạp |
| **Performance** | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| **Immutability** | ✅ | ✅ | ⚠️ (tự enforce) |
| **Private ctor** | ❌ | ❌ | ✅ |
| **Size** | < 64 bytes | Không giới hạn | Không giới hạn |
| **Inheritance** | ❌ | ✅ (sealed) | ✅ |

### 3.5. Recommendations từ Dự án

#### Quy tắc Thumb

```csharp
// 1-2 properties + Không cần EF Core → readonly record struct
public readonly record struct Actor(ActorKind Kind, string Value);

// 2-3 properties + Không cần EF Core → record
public sealed record Money(decimal Amount, string Currency);

// 3+ properties HOẶC EF Core HOẶC complex validation → class
public class Address { ... }
```

#### Trade-offs cần cân nhắc

| Muốn | Chấp nhận | Chọn |
|------|----------|------|
| Performance tốt nhất | Limited features | `readonly record struct` |
| Ngắn gọn, immutable | EF Core limitations | `record` |
| EF Core + flexible | Verbose | `class` |

#### Pattern kết hợp

```csharp
// Dùng record cho VO đơn giản trong domain
public sealed record Money(decimal Amount, string Currency);

// Dùng class cho complex VO cần EF Core
public class Order
{
    public Money TotalAmount { get; set; }  // EF Core owned entity
    public Address ShippingAddress { get; set; }  // Complex VO
}
```

> **Lưu ý**: EF Core 8+ có thể map `record` là Owned Entity, nhưng `class` vẫn linh hoạt hơn cho complex scenarios.

---

## 4. Best Practices trong Dự án

### 4.1. Factory Methods Pattern

Tất cả Value Object và Aggregate Root đều sử dụng **Factory Methods** thay vì public constructors:

```csharp
// Thay vì: new Address(...)
// Dùng: Address.Of(...)

// Thay vì: new OrderEntity { ... }
// Dùng: OrderEntity.Create(...)
```

**Lợi ích:**
- Validation tập trung
- Named parameters rõ ràng
- Có thể thay đổi implementation mà không ảnh hưởng call sites
- Ngăn chặn invalid state

### 4.2. Domain Events Pattern

Aggregate Root sử dụng Domain Events để giao tiếp:

```csharp
public void CancelOrder(string reason, string performBy)
{
    UpdateStatus(OrderStatus.Canceled, performBy);
    CancelReason = reason;
    AddDomainEvent(new OrderCancelledDomainEvent(this));
}
```

**Flow:**
1. Domain method thực hiện business logic
2. Gọi `AddDomainEvent()` để đăng ký event
3. Repository hoặc Unit of Work publish events sau khi save
4. Event handlers xử lý side effects (gửi email, cập nhật inventory, v.v.)

### 4.3. Encapsulation Pattern

Collections luôn được encapsulate:

```csharp
private readonly List<OrderItemEntity> _orderItems = new();
public IReadOnlyList<OrderItemEntity> OrderItems => _orderItems.AsReadOnly();

// Domain method để thay đổi collection
public void AddOrderItem(Product product, int quantity)
{
    // Validation
    var orderItem = OrderItemEntity.Create(...);
    _orderItems.Add(orderItem);
}
```

**Lợi ích:**
- Không thể thay đổi collection từ bên ngoài
- Business logic validation được thực hiện
- Invariants được bảo vệ

### 4.4. Audit Trail Pattern

Mọi thay đổi đều được ghi lại:

```csharp
public void UpdateShippingAddress(Address shippingAddress, string performBy)
{
    ShippingAddress = shippingAddress;
    LastModifiedBy = performBy;
    LastModifiedOnUtc = DateTimeOffset.UtcNow;
}
```

**Fields audit:**
- `CreatedBy`, `CreatedOnUtc`
- `LastModifiedBy`, `LastModifiedOnUtc`

---

## 5. Entity vs Value Object

### 5.1. Phân biệt trong Dự án

| Aspect | Entity | Value Object |
|--------|--------|--------------|
| **Identity** | Có ID riêng | Không có ID |
| **Equality** | By ID | By value |
| **Lifecycle** | Có lifecycle dài | Thay thế thay vì update |
| **Example** | OrderEntity, ProductEntity | Money, Address, OrderNo |

### 5.2. OrderItemEntity - Entity hay Value Object?

```csharp
public class OrderItemEntity : Entity<Guid>
{
    public Guid OrderId { get; set; }
    public Product Product { get; set; } = default!;
    public int Quantity { get; set; }
    public decimal LineTotal => Quantity * Product.Price;
}
```

**Quyết định:**
- Là **Entity** vì có `Id` riêng
- Cần reference từ OrderEntity
- Có thể cần track riêng (ví dụ: return specific item)
- Nhưng trong nhiều trường hợp, có thể thiết kế là Value Object nếu không cần reference riêng

---

## 6. Recommendations

### 6.1. Cải tiến có thể áp dụng

#### 6.1.1. Chuyển Value Object sang `init` properties

```csharp
// Hiện tại
public class Address
{
    public string City { get; set; } = default!;
}

// Cải tiến
public class Address
{
    public string City { get; init; } = default!;
}
```

**Lợi ích:** Giữ immutability semantic trong khi vẫn EF Core compatible.

#### 6.1.2. Sử dụng `required` keyword (C# 11+)

```csharp
public class Customer
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}
```

**Lợi ích:** Compiler enforce việc set giá trị khi khởi tạo.

#### 6.1.3. Value Object phức tạp nên override Equals và GetHashCode

```csharp
public class Address
{
    public override bool Equals(object? obj) => 
        obj is Address other && 
        AddressLine == other.AddressLine &&
        City == other.City && ...;

    public override int GetHashCode() => 
        HashCode.Combine(AddressLine, City, ...);
}
```

### 6.2. Anti-patterns cần tránh

1. **Không dùng public setter cho Aggregate Root ID**
   ```csharp
   // SAI
   public Guid Id { get; set; }
   
   // ĐÚNG
   public Guid Id { get; private set; }
   ```

2. **Không expose internal collection trực tiếp**
   ```csharp
   // SAI
   public List<OrderItemEntity> OrderItems { get; set; }
   
   // ĐÚNG
   private readonly List<OrderItemEntity> _orderItems = new();
   public IReadOnlyList<OrderItemEntity> OrderItems => _orderItems.AsReadOnly();
   ```

3. **Không để business logic leak ra Application Layer**
   ```csharp
   // SAI - Trong Application Layer
   order.Status = OrderStatus.Canceled;
   
   // ĐÚNG - Trong Domain Layer
   order.CancelOrder(reason, performedBy);
   ```

---

## 7. Kết luận

Dự án ProgCoder Shop Microservices triển khai Aggregate Root và Value Object theo đúng nguyên tắc Domain-Driven Design:

1. **Aggregate Root** sử dụng `sealed class` kế thừa từ `Aggregate<T>` base class
2. **Value Object** linh hoạt giữa `record` và `class` tùy use case
3. **Factory Methods** đảm bảo validation và encapsulation
4. **Domain Events** cho phép loose coupling giữa các bounded context
5. **Audit Trail** đảm bảo traceability

Cách triển khai này cân bằng giữa:
- **Domain purity**: Business logic tập trung trong Domain Layer
- **Technical constraints**: EF Core compatibility, serialization
- **Performance**: Lightweight value objects khi cần
- **Maintainability**: Rõ ràng, dễ test, dễ refactor

---

## Tài liệu Tham khảo

- [Microsoft: Implementing DDD](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice)
- [Vaughn Vernon: Implementing Domain-Driven Design](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577)
- [Eric Evans: Domain-Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)
- [C# Records](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/records)
