# Record trong C# - Hướng dẫn Toàn diện

**Author:** ProgCoder Shop Team  
**Updated:** January 2026  
**C# Version:** 9.0+, 10.0+, 11.0+, 12.0+

---

## Table of Contents

1. [Giới thiệu](#1-giới-thiệu)
2. [Cú pháp cơ bản](#2-cú-pháp-cơ-bản)
3. [Immutability](#3-immutability)
4. [Primary Constructor](#4-primary-constructor)
5. [Equality and Comparison](#5-equality-and-comparison)
6. [With Expressions](#6-with-expressions)
7. [Positional vs Property-Init Records](#7-positional-vs-property-init-records)
8. [Inheritance](#8-inheritance)
9. [Pattern Matching](#9-pattern-matching)
10. [Record Structs](#10-record-structs)
11. [Deconstruct](#11-deconstruct)
12. [Best Practices](#12-best-practices)
13. [Use Cases Thực tế](#13-use-cases-thực-tế)
14. [Performance Considerations](#14-performance-considerations)
15. [Migration từ Class sang Record](#15-migration-từ-class-sang-record)
16. [Common Pitfalls](#16-common-pitfalls)
17. [C# 12+ Updates](#17-c-12-updates)

---

## 1. Giới thiệu

### 1.1. Record là gì?

**Record** là type reference trong C# (từ C# 9.0) được thiết kế đặc biệt cho **immutable data models**.

```csharp
// Đơn giản và trực quan
public record Person(string Name, int Age);
```

### 1.2. Tại sao cần Record?

#### Trước C# 9 (Class truyền thống):

```csharp
public class Person
{
    public string Name { get; }  // Bắt buộc read-only
    public int Age { get; }

    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }

    // BẮT BUỘC override Equals(), GetHashCode() cho value semantics
    public override bool Equals(object obj) { /* 20+ dòng code */ }
    public override int GetHashCode() { /* 10+ dòng code */ }
    public static bool operator ==(Person left, Person right) { /* ... */ }
    public static bool operator !=(Person left, Person right) { /* ... */ }
}
```

#### Với Record (C# 9+):

```csharp
// Tất cả trong 1 dòng
public record Person(string Name, int Age);
```

### 1.3. Các tính năng chính

| Tính năng | Description |
|-----------|-------------|
| **Immutable** | Properties tự động read-only |
| **Value semantics** | So sánh theo giá trị, không theo reference |
| **Auto-generated** | Equals, GetHashCode, ==, != |
| **Primary constructor** | Constructor chính thân thiện |
| **With expression** | Tạo variant immutable |
| **Pattern matching** | Hỗ trợ deconstruct và matching |
| **Minimal boilerplate** | Giảm đáng kể code lặp lại |

---

## 2. Cú pháp cơ bản

### 2.1. Positional Record (Khuyên dùng)

```csharp
// Dạng ngắn gọn nhất
public record Person(string Name, int Age);

// Sử dụng
var person = new Person("John", 30);
Console.WriteLine(person.Name);  // John
Console.WriteLine(person.Age);   // 30
```

### 2.2. Property-Init Record

```csharp
public record Person
{
    public string Name { get; init; }
    public int Age { get; init; }
}

// Sử dụng
var person = new Person { Name = "John", Age = 30 };
```

### 2.3. Kết hợp

```csharp
public record Person(string Name, int Age)
{
    public string FullName => $"{Name} ({Age} years old)";
}

var person = new Person("John", 30);
Console.WriteLine(person.FullName);  // John (30 years old)
```

### 2.4. Sealed Record (Khuyên dùng cho DTOs)

```csharp
// Ngăn chặn inheritance - recommended cho DTOs/Commands/Queries
public sealed record CreateProductCommand(CreateProductDto Dto, Actor Actor);
```

---

## 3. Immutability

### 3.1. Properties tự động read-only

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);

// ❌ Compile error: Init-only property can only be assigned in constructor or initializer
// person.Name = "Jane";
// person.Age = 31;
```

### 3.2. Setters không được hỗ trợ

```csharp
// ❌ Không thể dùng { get; set; }
public record Person
{
    public string Name { get; set; }  // Compilation warning
}
```

### 3.3. Mutability chỉ qua with expression

```csharp
public record Person(string Name, int Age);

var person1 = new Person("John", 30);
var person2 = person1 with { Age = 31 };  // Tạo record mới

Console.WriteLine(person1.Age);  // 30 (original unchanged)
Console.WriteLine(person2.Age);  // 31 (new instance)
```

### 3.4. Khi nào cần mutable records?

**RARE CASE:** Chỉ dùng khi record có private mutable state:

```csharp
public record CacheItem(string Key, T Value)
{
    private DateTime _lastAccessed = DateTime.UtcNow;
    
    public DateTime LastAccessed => _lastAccessed;
    
    public void Touch() => _lastAccessed = DateTime.UtcNow;
}
```

**Warning:** Avoid trừ khi có lý do chính đáng.

---

## 4. Primary Constructor

### 4.1. Cú pháp cơ bản

```csharp
public record Person(string Name, int Age)
{
    // Name và Age tự động trở thành properties
}

// Equivalent với:
public class Person
{
    public string Name { get; }
    public int Age { get; }
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }
}
```

### 4.2. Validation trong constructor

```csharp
public record Person(string Name, int Age)
{
    public Person(string name, int age) : this(name, age)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        if (age < 0)
            throw new ArgumentException("Age cannot be negative", nameof(age));
    }
}
```

### 4.3. Private constructor

```csharp
public record Person(string Name, int Age)
{
    private Person(string name, int age) : this(name, age) { }
    
    public static Person Create(string name, int age)
    {
        // Validation logic
        return new Person(name, age);
    }
}
```

### 4.4. Constructor parameters không thành properties

```csharp
public record Person(string Name, int Age, string _privateField)
{
    // _privateField KHÔNG trở thành property
    // Chỉ Name và Age là properties
}

var person = new Person("John", 30, "secret");
Console.WriteLine(person.Name);  // John
// Console.WriteLine(person._privateField);  // ❌ Compile error
```

---

## 5. Equality and Comparison

### 5.1. Value-based Equality

```csharp
public record Person(string Name, int Age);

var p1 = new Person("John", 30);
var p2 = new Person("John", 30);
var p3 = new Person("Jane", 30);

Console.WriteLine(p1 == p2);  // True (equal values)
Console.WriteLine(p1.Equals(p2));  // True
Console.WriteLine(p1 == p3);  // False (different name)
```

### 5.2. Reference Equality

```csharp
var p1 = new Person("John", 30);
var p2 = p1;  // Same reference

Console.WriteLine(ReferenceEquals(p1, p2));  // True
Console.WriteLine(p1 == p2);  // True (also value equality)
```

### 5.3. Nested Records (Deep Equality)

```csharp
public record Address(string City, string Country);
public record Person(string Name, Address HomeAddress);

var p1 = new Person("John", new Address("Hanoi", "Vietnam"));
var p2 = new Person("John", new Address("Hanoi", "Vietnam"));

Console.WriteLine(p1 == p2);  // True (deep comparison)
```

### 5.4. Type Equality

```csharp
public record Person(string Name, int Age);
public record Employee(string Name, int Age, string Dept) : Person(Name, Age);

Person p1 = new Employee("John", 30, "IT");
Person p2 = new Person("John", 30);

Console.WriteLine(p1 == p2);  // False (different types)
```

### 5.5. Custom Equality

```csharp
public record Person(string Name, int Age)
{
    public virtual bool Equals(Person? other)
    {
        if (other is null) return false;
        
        // Custom: Case-insensitive name comparison
        return Name.Equals(other.Name, StringComparison.OrdinalIgnoreCase) &&
               Age == other.Age;
    }
    
    public override int GetHashCode()
    {
        return HashCode.Combine(Name.ToUpperInvariant(), Age);
    }
}

var p1 = new Person("John", 30);
var p2 = new Person("JOHN", 30);

Console.WriteLine(p1 == p2);  // True (custom logic)
```

### 5.6. Null Comparison

```csharp
Person? p1 = new Person("John", 30);
Person? p2 = null;

Console.WriteLine(p1 == null);  // False
Console.WriteLine(null == p2);  // True
Console.WriteLine(p1 == p2);  // False
```

---

## 6. With Expressions

### 6.1. Basic Usage

```csharp
public record Person(string Name, int Age);

var original = new Person("John", 30);
var modified = original with { Age = 31 };

Console.WriteLine(original.Age);  // 30 (unchanged)
Console.WriteLine(modified.Age);  // 31 (new value)
Console.WriteLine(original == modified);  // False
```

### 6.2. Multiple Property Changes

```csharp
var original = new Person("John", 30);
var modified = original with { Name = "Jane", Age = 25 };

Console.WriteLine(modified);  // Person { Name = Jane, Age = 25 }
```

### 6.3. Nested With Expressions

```csharp
public record Address(string City, string Country);
public record Person(string Name, Address HomeAddress);

var original = new Person("John", new Address("Hanoi", "Vietnam"));
var modified = original with 
{ 
    Name = "Jane",
    HomeAddress = original.HomeAddress with { City = "Ho Chi Minh" }
};

Console.WriteLine(modified.HomeAddress.City);  // Ho Chi Minh
```

### 6.4. Use Case: Immutable Update

```csharp
public record CartItem(string ProductId, int Quantity, decimal Price);

var cart = new[]
{
    new CartItem("P1", 2, 10.99m),
    new CartItem("P2", 1, 5.99m)
};

// Update quantity of first item
var updatedCart = cart.Select(item => 
    item.ProductId == "P1" 
        ? item with { Quantity = item.Quantity + 1 }
        : item
).ToArray();

Console.WriteLine(updatedCart[0].Quantity);  // 3
```

---

## 7. Positional vs Property-Init Records

### 7.1. Positional Records (Primary Constructor)

```csharp
public record Person(string Name, int Age);

// Sử dụng
var person = new Person("John", 30);
```

**Ưu điểm:**
- ✅ Ngắn gọn, 1 dòng
- ✅ Properties tự động từ constructor parameters
- ✅ Hỗ trợ deconstruct
- ✅ Phù hợp cho DTOs, Commands, Queries

**Nhược điểm:**
- ❌ Không thể có default values cho properties
- ❌ Tất cả parameters đều thành public properties

### 7.2. Property-Init Records

```csharp
public record Person
{
    public string Name { get; init; }
    public int Age { get; init; }
    
    public Person() { }  // Optional parameterless constructor
}

// Sử dụng
var person = new Person { Name = "John", Age = 30 };
```

**Ưu điểm:**
- ✅ Có thể có default values
- ✅ Có thể có private properties
- ✅ Linh hoạt hơn với complex initialization

**Nhược điểm:**
- ❌ Verboser syntax
- ❌ Không hỗ trợ deconstruct tự động

### 7.3. Mixed Approach (Recommended cho Complex Types)

```csharp
public record Person(string Name, int Age)
{
    public string FullName => $"{Name} ({Age} years old)";
    private DateTime _created = DateTime.UtcNow;
}

// Sử dụng
var person = new Person("John", 30);
Console.WriteLine(person.FullName);  // John (30 years old)
```

---

## 8. Inheritance

### 8.1. Basic Inheritance

```csharp
public record Person(string Name, int Age);
public record Employee(string Name, int Age, string Department) : Person(Name, Age);

var emp = new Employee("John", 30, "IT");
Console.WriteLine(emp.Name);  // John
Console.WriteLine(emp.Department);  // IT
```

### 8.2. Sealed Base Classes

```csharp
// Ngăn chặn inheritance
public sealed record Person(string Name, int Age);

// ❌ Compile error: Cannot derive from sealed type
// public record Employee(string Name, int Age, string Department) : Person(Name, Age);
```

### 8.3. Override Equals for Derived Classes

```csharp
public record Person(string Name, int Age)
{
    public virtual bool Equals(Person? other) => /* custom logic */;
}

public record Employee(string Name, int Age, string Department) : Person(Name, Age)
{
    // Can override equality to include Department
    public sealed override bool Equals(Person? other)
    {
        if (other is not Employee emp) return false;
        return base.Equals(other) && Department == emp.Department;
    }
}
```

### 8.4. Virtual vs Sealed Methods

```csharp
public record Person(string Name, int Age)
{
    // Virtual: Derived classes can override
    public virtual string GetDescription() => $"Person: {Name}";
}

public record Employee(string Name, int Age, string Department) : Person(Name, Age)
{
    // Override to provide specific behavior
    public override string GetDescription() => $"Employee: {Name} in {Department}";
}
```

---

## 9. Pattern Matching

### 9.1. Type Pattern

```csharp
public record Person(string Name, int Age);
public record Employee(string Name, int Age, string Department) : Person(Name, Age);

Person p = new Employee("John", 30, "IT");

if (p is Employee emp)
{
    Console.WriteLine(emp.Department);  // IT
}
```

### 9.2. Property Pattern

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);

if (person is { Name: "John", Age: >= 18 })
{
    Console.WriteLine("John is an adult");
}
```

### 9.3. Deconstruct Pattern

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);

// Deconstruct vào variables
var (name, age) = person;
Console.WriteLine(name);  // John
Console.WriteLine(age);   // 30
```

### 9.4. Positional Pattern

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);

var message = person switch
{
    ("John", var age) => $"John is {age} years old",
    (_, var age) when age >= 18 => "An adult",
    _ => "A minor"
};

Console.WriteLine(message);  // John is 30 years old
```

### 9.5. Relational Pattern

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);

var category = person switch
{
    { Age: < 0 } => "Invalid age",
    { Age: < 18 } => "Minor",
    { Age: < 65 } => "Adult",
    _ => "Senior"
};

Console.WriteLine(category);  // Adult
```

---

## 10. Record Structs

### 10.1. Cú pháp cơ bản (C# 10+)

```csharp
// Value type record
public record struct Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = new Point(1, 2);

Console.WriteLine(p1 == p2);  // True (value semantics)
```

### 10.2. Khác biệt giữa record và record struct

| Aspect | record (class) | record struct |
|--------|----------------|---------------|
| Type | Reference type | Value type |
| Nullability | Can be null | Cannot be null (unless Nullable<Point>) |
| Memory allocation | Heap allocation | Stack allocation |
| Copying | Reference copy | Value copy |
| Default value | null | default(Point) |
| With expression | ✅ Yes | ✅ Yes |
| Equality | Value-based | Value-based |

### 10.3. Read-only Record Structs (C# 10+)

```csharp
// Immutable value type
public readonly record struct Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = p1 with { X = 3 };  // Creates new struct

Console.WriteLine(p1.X);  // 1 (unchanged)
Console.WriteLine(p2.X);  // 3 (new value)
```

### 10.4. Use Cases cho Record Structs

```csharp
// ID wrapper (small, frequently compared)
public readonly record struct ProductId(Guid Value);

public record Product(ProductId Id, string Name);

var product = new Product(new ProductId(Guid.NewGuid()), "Laptop");
Console.WriteLine(product.Id.Value);
```

**Ưu điểm:**
- Stack-allocated (fast, no GC pressure)
- Immutable
- Value-based equality
- Perfect cho small, frequently-used types

---

## 11. Deconstruct

### 11.1. Automatic Deconstruct (Positional Records)

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);

// Deconstruct
var (name, age) = person;
Console.WriteLine(name);  // John
Console.WriteLine(age);   // 30
```

### 11.2. Manual Deconstruct (Property-Init Records)

```csharp
public record Person
{
    public string Name { get; init; }
    public int Age { get; init; }
    
    // Manual deconstruct method
    public void Deconstruct(out string name, out int age)
    {
        name = Name;
        age = Age;
    }
}

var person = new Person { Name = "John", Age = 30 };
var (name, age) = person;
```

### 11.3. Partial Deconstruct

```csharp
public record Person(string Name, int Age, string Email);

var person = new Person("John", 30, "john@example.com");

// Discard age with _
var (name, _, email) = person;
Console.WriteLine(name);  // John
Console.WriteLine(email); // john@example.com
```

### 11.4. Use Case: Tuple Assignment

```csharp
public record Point(int X, int Y);

var points = new[] { new Point(1, 2), new Point(3, 4) };

// Destructuring in LINQ
var coordinates = points.Select(p => (p.X, p.Y)).ToArray();

Console.WriteLine(coordinates[0].X);  // 1
Console.WriteLine(coordinates[0].Y);  // 2
```

---

## 12. Best Practices

### 12.1. Khi nào dùng record?

**✅ NÊN DÙNG:**

```csharp
// DTOs (Data Transfer Objects)
public record CustomerDto(Guid Id, string Name, string Email);

// CQRS Commands
public sealed record CreateProductCommand(CreateProductDto Dto, Actor Actor);

// CQRS Queries
public sealed record GetProductByIdQuery(Guid ProductId) : IQuery<GetProductByIdResult>;

// CQRS Results
public record GetProductByIdResult(ProductDto Product);

// Configuration
public record DatabaseOptions(string ConnectionString, int MaxPoolSize);

// Value Objects (DDD)
public readonly record struct Money(decimal Amount, string Currency);

// Domain Events
public sealed record OrderCreatedEvent(Guid OrderId, DateTime CreatedAt);

// API Response Models
public record ApiResponse<T>(T Data, string Message, bool Success);
```

**❌ KHÔNG NÊN DÙNG:**

```csharp
// ❌ Entities with behavior
public record Order()  // Bad: Should be class with methods
{
    public void AddItem(OrderItem item) { }
}

// ❌ Services
public record CustomerService()  // Bad: Should be class with injected dependencies
{
    public Task<Customer> GetByIdAsync(Guid id) { }
}

// ❌ Mutable state
public record Counter()  // Bad: Records should be immutable
{
    public int Count { get; set; }  // ❌ Mutable property
}
```

### 12.2. Naming Conventions

```csharp
// ✅ Good: PascalCase for record name
public record Person(string Name, int Age);

// ✅ Good: PascalCase for properties
public record Customer(string CustomerName, string CustomerEmail);

// ✅ Good: Suffix for DTOs
public record CustomerDto(...);
public record CreateCustomerCommand(...);

// ✅ Good: Use sealed when no inheritance needed
public sealed record CreateProductCommand(...);

// ❌ Bad: camelCase for record name
public record person(string name, int age);  // ❌ Bad
```

### 12.3. Sealed vs Non-sealed

```csharp
// ✅ Sealed: Recommended cho DTOs, Commands, Queries
public sealed record CreateProductCommand(...);

// ✅ Sealed: Recommended cho final implementations
public sealed record CustomerDto(...);

// ✅ Non-sealed: When inheritance is needed
public record Person(string Name, int Age);
public record Employee(string Name, int Age, string Dept) : Person(Name, Age);
```

### 12.4. Default Values

```csharp
// ✅ Good: Property-init records with default values
public record Person
{
    public string Name { get; init; } = string.Empty;
    public int Age { get; init; } = 0;
}

// ✅ Good: Static factory methods for complex defaults
public record Person(string Name, int Age)
{
    public static Person CreateDefault() => new Person("Unknown", 0);
}
```

### 12.5. Validation

```csharp
// ✅ Good: Constructor validation
public record Person(string Name, int Age)
{
    public Person(string name, int age) : this(name, age)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
    }
}

// ✅ Good: Factory method validation
public record Person(string Name, int Age)
{
    public static Person Create(string name, int age)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        return new Person(name, age);
    }
}
```

---

## 13. Use Cases Thực tế

### 13.1. CQRS Pattern

```csharp
// Commands
public sealed record CreateProductCommand(CreateProductDto Dto, Actor Actor) : ICommand<Guid>;
public sealed record UpdateProductCommand(Guid Id, UpdateProductDto Dto, Actor Actor) : ICommand;
public sealed record DeleteProductCommand(Guid Id, Actor Actor) : ICommand;

// Queries
public sealed record GetProductByIdQuery(Guid ProductId) : IQuery<GetProductByIdResult>;
public sealed record GetAllProductsQuery(int Page, int PageSize) : IQuery<GetAllProductsResult>;

// Results
public record GetProductByIdResult(ProductDto Product);
public record GetAllProductsResult(IReadOnlyList<ProductDto> Products, int TotalCount);

// Handlers
public sealed class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, Guid>
{
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken ct)
    {
        // Command là immutable - không thể thay đổi accidentally
        var entity = mapper.Map<ProductEntity>(command.Dto);
        await repository.AddAsync(entity, ct);
        return entity.Id;
    }
}
```

### 13.2. API Response Models

```csharp
public record ApiResponse<T>(T Data, string Message, bool Success);

// Success response
var success = new ApiResponse<UserDto>(userDto, "User created successfully", true);

// Error response
var error = new ApiResponse<UserDto>(null, "Invalid input", false);

// Empty response
var empty = new ApiResponse<string>(string.Empty, "No data", false);
```

### 13.3. Configuration

```csharp
public record DatabaseOptions(
    string ConnectionString,
    int MaxPoolSize = 100,
    int CommandTimeout = 30
);

public record RedisOptions(
    string ConnectionString,
    int Database = 0,
    TimeSpan? Expiration = null
);

public record AppOptions(
    DatabaseOptions Database,
    RedisOptions Redis,
    string Environment
);
```

### 13.4. Value Objects (DDD)

```csharp
// Money value object
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new ArgumentException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
}

// Email value object
public sealed record Email(string Value)
{
    public Email(string value) : this(value)
    {
        if (!IsValidEmail(value))
            throw new ArgumentException("Invalid email format", nameof(value));
    }
    
    private static bool IsValidEmail(string email)
    {
        // Email validation logic
        return Regex.IsMatch(email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
    }
}
```

### 13.5. Domain Events

```csharp
public abstract record DomainEvent(Guid EventId, DateTime OccurredAt);

public sealed record OrderCreatedEvent(
    Guid OrderId, 
    Guid CustomerId, 
    DateTime OccurredAt) : DomainEvent(Guid.NewGuid(), OccurredAt);

public sealed record OrderStatusChangedEvent(
    Guid OrderId, 
    OrderStatus OldStatus, 
    OrderStatus NewStatus, 
    DateTime OccurredAt) : DomainEvent(Guid.NewGuid(), OccurredAt);
```

### 13.6. Immutable Collections

```csharp
public record ShoppingCart(IReadOnlyList<CartItem> Items)
{
    public ShoppingCart AddItem(CartItem newItem)
    {
        var updatedItems = Items.Append(newItem).ToList();
        return this with { Items = updatedItems };
    }
    
    public ShoppingCart RemoveItem(Guid productId)
    {
        var updatedItems = Items.Where(item => item.ProductId != productId).ToList();
        return this with { Items = updatedItems };
    }
}

var cart = new ShoppingCart(new List<CartItem>());
cart = cart.AddItem(new CartItem(Guid.NewGuid(), "Laptop", 999.99m));
```

---

## 14. Performance Considerations

### 14.1. Memory Allocation

#### **Record (class):**

```csharp
public record Person(string Name, int Age);

var person = new Person("John", 30);
// - Allocated on heap
// - Subject to GC
// - Reference copy is fast (just pointer)
```

#### **Record struct:**

```csharp
public record struct Point(int X, int Y);

var point = new Point(1, 2);
// - Allocated on stack (or embedded in containing type)
// - No GC pressure
// - Value copy copies all fields
```

### 14.2. Equality Performance

```csharp
// Record equality is O(n) where n = number of properties
public record Person(string Name, int Age, Address Address);
// Equality comparison checks: Name, Age, Address (recursively)

// Optimization: Use record struct for small, frequently-compared types
public readonly record struct ProductId(Guid Value);
// - Fast equality (just compare Guid)
// - No heap allocation
// - Perfect for dictionary keys
```

### 14.3. GetHashCode() Performance

```csharp
// GetHashCode() combines hash codes of ALL properties
// Can be expensive for records with many properties

public record LargeRecord(
    string Prop1, string Prop2, string Prop3,
    string Prop4, string Prop5, string Prop6,
    string Prop7, string Prop8, string Prop9,
    string Prop10
);
// GetHashCode() is O(n) where n = number of properties
```

**Optimization:** Override GetHashCode() cho complex records:

```csharp
public record Person(string Name, int Age, Address Address)
{
    public override int GetHashCode()
    {
        // Only combine primary properties
        return HashCode.Combine(Name, Age);
    }
}
```

### 14.4. With Expression Performance

```csharp
var original = new Person("John", 30, new Address("Hanoi", "Vietnam"));
var modified = original with { Name = "Jane" };

// With expression creates NEW instance
// Copies ALL properties from original
// O(n) where n = number of properties

// For large records, consider builder pattern instead:
var modified = original.WithName("Jane");  // Custom method
```

### 14.5. Performance Benchmarks (Approximate)

| Operation | Record (class) | Record struct | Class (custom) |
|-----------|----------------|---------------|----------------|
| Allocation | Heap (slower) | Stack (fast) | Heap (slower) |
| Equality check | O(n) | O(n) | O(1) reference |
| GetHashCode | O(n) | O(n) | O(1) reference |
| With expression | O(n) | O(n) | N/A |
| GC pressure | Yes | No | Yes |

---

## 15. Migration từ Class sang Record

### 15.1. Simple DTO Class

**Trước (Class):**
```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public override bool Equals(object obj)
    {
        if (obj is not Person other) return false;
        return Name == other.Name && Age == other.Age;
    }
    
    public override int GetHashCode()
    {
        return HashCode.Combine(Name, Age);
    }
}
```

**Sau (Record):**
```csharp
// ✅ Tự động có tất cả!
public sealed record Person(string Name, int Age);
```

### 15.2. Class với Constructor Logic

**Trước (Class):**
```csharp
public class Person
{
    public string Name { get; }
    public int Age { get; }
    
    public Person(string name, int age)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        if (age < 0)
            throw new ArgumentException("Age cannot be negative", nameof(age));
        
        Name = name;
        Age = age;
    }
    
    // ... Equals, GetHashCode implementations
}
```

**Sau (Record):**
```csharp
public sealed record Person(string Name, int Age)
{
    public Person(string name, int age) : this(name, age)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        if (age < 0)
            throw new ArgumentException("Age cannot be negative", nameof(age));
    }
}
```

### 15.3. Class với Additional Properties

**Trước (Class):**
```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public string FullName => $"{Name} ({Age} years old)";
}
```

**Sau (Record):**
```csharp
public sealed record Person(string Name, int Age)
{
    public string FullName => $"{Name} ({Age} years old)";
}
```

### 15.4. Migration Checklist

- [ ] Remove private fields (use primary constructor)
- [ ] Remove manual `Equals()`, `GetHashCode()`, `==`, `!=`
- [ ] Remove `set` on properties (use immutability)
- [ ] Replace `new ClassName { ... }` with constructor syntax
- [ ] Add `sealed` if no inheritance needed
- [ ] Update unit tests to use value equality
- [ ] Verify serialization/deserialization works

---

## 16. Common Pitfalls

### 16.1. Mutating Record After Creation

```csharp
// ❌ Pitfall: Trying to mutate record
public record Person(string Name, int Age);

var person = new Person("John", 30);
person.Name = "Jane";  // ❌ Compile error

// ✅ Solution: Use with expression
var updatedPerson = person with { Name = "Jane" };
```

### 16.2. Not Overriding GetHashCode() After Equals()

```csharp
// ❌ Pitfall: Breaks contract
public record Person(string Name, int Age)
{
    public virtual bool Equals(Person? other) => /* custom logic */;
    // ❌ Forgot to override GetHashCode()
}

// ✅ Solution: Always override both
public record Person(string Name, int Age)
{
    public virtual bool Equals(Person? other) => /* custom logic */;
    public override int GetHashCode() => HashCode.Combine(Name, Age);
}
```

### 16.3. Inheritance Confusion

```csharp
// ❌ Pitfall: Type equality confusion
public record Person(string Name, int Age);
public record Employee(string Name, int Age, string Dept) : Person(Name, Age);

Person p1 = new Employee("John", 30, "IT");
Person p2 = new Person("John", 30);

Console.WriteLine(p1 == p2);  // False (different types)

// ✅ Solution: Be aware of type equality
```

### 16.4. Reference Equality Expectation

```csharp
// ❌ Pitfall: Expecting reference equality
var list = new List<Person> { new Person("John", 30) };
var person = list[0];

Console.WriteLine(list.Contains(person));  // True (value equality)
Console.WriteLine(ReferenceEquals(list[0], person));  // True (same reference)

var person2 = new Person("John", 30);
Console.WriteLine(list.Contains(person2));  // True (value equality!)
```

### 16.5. Serialization Issues

```csharp
// ❌ Pitfall: Some serializers require parameterless constructor
[Serializable]
public record Person(string Name, int Age);

// ✅ Solution: Use property-init records or explicit constructor
[Serializable]
public record Person
{
    public string Name { get; init; }
    public int Age { get; init; }
}

// Or use explicit serializer attributes
public record Person(string Name, int Age)
{
    [JsonConstructor]
    public Person(string name, int age) : this(name, age) { }
}
```

### 16.6. Circular References

```csharp
// ❌ Pitfall: Circular references cause infinite recursion in Equals()
public record Node(int Value, Node? Left, Node? Right);

var left = new Node(1, null, null);
var right = new Node(3, null, null);
var root = new Node(2, left, right);

left = left with { Right = root };  // ❌ Circular!

// ✅ Solution: Use identity-based equality for nodes with cycles
public record Node(int Id, int Value, Node? Left, Node? Right)
{
    public virtual bool Equals(Node? other)
    {
        if (other is null) return false;
        return Id == other.Id;  // Identity-based equality
    }
}
```

---

## 17. C# 12+ Updates

### 17.1. Primary Constructors for Classes (C# 12)

```csharp
// Classes now support primary constructors too
public class Person(string Name, int Age)
{
    public void Print()
    {
        Console.WriteLine($"{Name} is {Age} years old");
    }
}

var person = new Person("John", 30);
person.Print();
```

### 17.2. Alias Any Type (C# 12)

```csharp
// Type aliases for records
using PersonDto = (string Name, int Age);

var person: PersonDto = ("John", 30);
```

### 17.3. Experimental Features

Các experimental features có thể được thêm vào các bản C# tương lai:
- `record class` vs `record struct` explicit syntax
- Enhanced pattern matching for records
- Default interface methods for records

---

## 18. Summary

### 18.1. Key Takeaways

1. **Record = Immutable Data Models**
   - Tự động immutable
   - Value-based equality
   - Minimal boilerplate

2. **Use Cases**
   - DTOs, Commands, Queries
   - Configuration
   - Value Objects
   - Domain Events

3. **Best Practices**
   - Dùng `sealed` khi không cần inheritance
   - Dùng positional records cho đơn giản
   - Override `Equals()`/`GetHashCode()` pairs
   - Validate trong constructor

4. **Performance**
   - Record class: Heap allocation
   - Record struct: Stack allocation
   - Equality: O(n) comparison
   - Perfect cho small, frequently-used types

5. **Avoid**
   - Entities with behavior (use class)
   - Services (use class)
   - Mutable state (use class)

### 18.2. Decision Tree

```
Need a type for:
├─ DTOs/Commands/Queries?
│  └─> Use sealed record
├─ Configuration?
│  └─> Use record
├─ Value Objects?
│  └─> Use readonly record struct (small) or record (complex)
├─ Domain Events?
│  └─> Use sealed record
├─ Entities with behavior?
│  └─> Use class (not record)
├─ Services?
│  └─> Use class (not record)
└─ Mutable state?
   └─> Use class (not record)
```

---

## References

- [C# Records Documentation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record)
- [C# 9.0 Records](https://devblogs.microsoft.com/dotnet/welcome-to-c-9-0/)
- [C# 10.0 Record Structs](https://devblogs.microsoft.com/dotnet/csharp-10/)
- [C# 12.0 Primary Constructors](https://devblogs.microsoft.com/dotnet/csharp-12/)

---

## Appendices

### A. Comparison Table: Record vs Class vs Struct

| Feature | Record (class) | Record struct | Class (custom) | Struct |
|---------|----------------|---------------|----------------|--------|
| Type | Reference | Value | Reference | Value |
| Immutability | Auto | Auto | Manual | Manual |
| Equality | Value-based | Value-based | Reference-based | Reference-based |
| Auto Equals/GetHashCode | ✅ | ✅ | ❌ | ❌ |
| Nullability | Can be null | Cannot be null | Can be null | Cannot be null |
| Allocation | Heap | Stack | Heap | Stack |
| GC pressure | Yes | No | Yes | No |
| Primary constructor | ✅ | ✅ | ✅ (C#12+) | ✅ (C#12+) |
| Inheritance | Single | No | Single | No |
| With expression | ✅ | ✅ | ❌ | ❌ |

### B. Code Templates

#### Template 1: DTO
```csharp
public sealed record CustomerDto(
    Guid Id,
    string Name,
    string Email,
    DateTime CreatedAt
);
```

#### Template 2: CQRS Command
```csharp
public sealed record CreateCustomerCommand(
    CreateCustomerDto Dto,
    Actor Actor
) : ICommand<Guid>;
```

#### Template 3: CQRS Query
```csharp
public sealed record GetCustomerByIdQuery(
    Guid CustomerId
) : IQuery<GetCustomerByIdResult>;
```

#### Template 4: CQRS Result
```csharp
public record GetCustomerByIdResult(CustomerDto Customer);
```

#### Template 5: Value Object
```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new ArgumentException("Cannot add different currencies");
        return new Money(left.Amount + right.Amount, left.Currency);
    }
}
```

#### Template 6: Configuration
```csharp
public record DatabaseOptions(
    string ConnectionString,
    int MaxPoolSize = 100,
    int CommandTimeout = 30,
    bool EnableSensitiveDataLogging = false
);
```

#### Template 7: Domain Event
```csharp
public sealed record CustomerCreatedEvent(
    Guid CustomerId,
    string CustomerName,
    DateTime OccurredAt
) : DomainEvent(Guid.NewGuid(), OccurredAt);
```

---

**End of Document**
