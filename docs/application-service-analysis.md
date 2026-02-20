# PHÂN TÍCH APPLICATION SERVICE TRONG CLEAN ARCHITECTURE

## ProgCoder Shop Microservices

---

## Mục Lục

- [A. Tổng Quan Kiến Trúc](#a-tổng-quan-kiến-trúc)
- [B. Các Thành Phần Chính của Application Layer](#b-các-thành-phần-chính-củả-application-layer)
  - [1. CQRS Core Interfaces (BuildingBlocks.CQRS)](#1-cqrs-core-interfaces-buildingblockscqrs)
  - [2. Commands và Queries](#2-commands-và-queries)
  - [3. Validators (FluentValidation)](#3-validators-fluentvalidation)
  - [4. Dependency Injection Configuration](#4-dependency-injection-configuration)
  - [5. Repository Pattern](#5-repository-pattern)
  - [6. Unit of Work Pattern](#6-unit-of-work-pattern)
  - [7. Application Services](#7-application-services)
  - [8. Exception Handling](#8-exception-handling)
- [C. Cấu Trúc Thư Mục Application Layer](#c-cấu-trúc-thư-mục-application-layer)
- [D. Design Patterns Sử Dụng](#d-design-patterns-sử-dụng)
- [E. Coding Conventions](#e-coding-conventions)
- [F. Ví Dụ Hoàn Chỉnh](#f-ví-dụ-hoàn-chỉnh)
  - [1. CreateProductCommand](#1-createproductcommand)
  - [2. GetProductByIdQuery](#2-getproductbyidquery)
  - [3. DependencyInjection Configuration](#3-dependencyinjection-configuration)
- [G. Kết Luận](#g-kết-luận)

---

## A. Tổng Quan Kiến Trúc

Dự án sử dụng **Clean Architecture + CQRS Pattern** với **MediatR** làm messaging mediator. Application Layer đóng vai trò là lớp trung gian giữa Presentation Layer và Domain Layer, chứa business logic và orchestration.

### Các đặc điểm chính:
- **.NET 8** với C# 12 (Primary constructors, Records)
- **MediatR** cho CQRS implementation
- **FluentValidation** cho input validation
- **AutoMapper** cho object mapping
- **Unit of Work Pattern** cho transaction management
- **Repository Pattern** cho data access abstraction

---

## B. Các Thành Phần Chính của Application Layer

### 1. CQRS Core Interfaces (BuildingBlocks.CQRS)

**File:** `Shared/BuildingBlocks/CQRS/`

```csharp
// ICommand.cs - Định nghĩa Command interfaces
public interface ICommand : ICommand<Unit>
public interface ICommand<out TResponse> : IRequest<TResponse>

// ICommandHandler.cs - Định nghĩa Command Handler interfaces
public interface ICommandHandler<in TCommand> : ICommandHandler<TCommand, Unit>
public interface ICommandHandler<in TCommand, TResponse> : IRequestHandler<TCommand, TResponse>

// IQuery.cs - Định nghĩa Query interfaces
public interface IQuery<out TResponse> : IRequest<TResponse>

// IQueryHandler.cs - Định nghĩa Query Handler interfaces
public interface IQueryHandler<in TQuery, TResponse> : IRequestHandler<TQuery, TResponse>
```

**Đặc điểm:**
- Kế thừa từ MediatR's `IRequest` và `IRequestHandler`
- Sử dụng C# 9.0 `record` types cho Commands và Queries
- Generic constraints đảm bảo type safety

---

### 2. Commands và Queries

**Tổng số:** 87 Commands và Queries được định nghĩa trong các service.

#### 2.1. Command Pattern

```csharp
// CreateProductCommand.cs - Ví dụ Command
namespace Catalog.Application.Features.Product.Commands;

public record CreateProductCommand(CreateProductDto Dto, Actor Actor) : ICommand<Guid>;

// Validator đi kèm
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Dto.Name).NotEmpty().WithMessage(MessageCode.ProductNameIsRequired);
        RuleFor(x => x.Dto.Sku).NotEmpty().WithMessage(MessageCode.SkuIsRequired);
    }
}

// Handler sử dụng Primary Constructor (C# 12)
public class CreateProductCommandHandler(IDocumentSession session) : ICommandHandler<CreateProductCommand, Guid>
{
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken ct)
    {
        var product = new ProductEntity
        {
            Name = command.Dto.Name,
            Sku = command.Dto.Sku,
            // ...
        };
        
        session.Store(product);
        await session.SaveChangesAsync(ct);
        
        return product.Id;
    }
}
```

#### 2.2. Query Pattern

```csharp
// GetProductByIdQuery.cs - Ví dụ Query
namespace Catalog.Application.Features.Product.Queries;

public sealed record GetProductByIdQuery(Guid ProductId) : IQuery<GetProductByIdResult>;

public sealed class GetProductByIdQueryHandler(IDocumentSession session, IMapper mapper) 
    : IQueryHandler<GetProductByIdQuery, GetProductByIdResult>
{
    public async Task<GetProductByIdResult> Handle(GetProductByIdQuery query, CancellationToken ct)
    {
        var product = await session.Query<ProductEntity>()
            .FirstOrDefaultAsync(x => x.Id == query.ProductId, ct);
            
        if (product == null)
            throw new NotFoundException(MessageCode.ProductNotFound);
            
        return new GetProductByIdResult(mapper.Map<ProductDto>(product));
    }
}

// Result DTO
public sealed record GetProductByIdResult(ProductDto Item);
```

#### 2.3. Phân bố theo Service

| Service | Commands | Queries | Tổng |
|---------|----------|---------|------|
| Basket | 3 | 1 | 4 |
| Catalog | 12 | 13 | 25 |
| Discount | 9 | 7 | 16 |
| Inventory | 15 | 12 | 27 |
| Notification | 3 | 9 | 12 |
| Order | 4 | 19 | 23 |
| Report | 3 | 3 | 6 |
| Search | 2 | 1 | 3 |
| **Tổng** | **51** | **65** | **116** |

---

### 3. Validators (FluentValidation)

**Tổng số:** 44 Validators - mỗi Command/Query đều có Validator tương ứng.

#### 3.1. Pattern Validation

```csharp
public sealed class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Dto.Name)
            .NotEmpty()
            .WithMessage(MessageCode.ProductNameIsRequired);
            
        RuleFor(x => x.Dto.Sku)
            .NotEmpty()
            .WithMessage(MessageCode.SkuIsRequired);
            
        RuleFor(x => x.Dto.Price)
            .GreaterThan(0)
            .WithMessage(MessageCode.PriceMustBeGreaterThanZero);
    }
}
```

#### 3.2. Quy tắc Validation

- Sử dụng `MessageCode` constants thay vì hard-coded strings
- `ValidationBehavior` tự động chạy trước Handler thông qua MediatR pipeline
- Chỉ dùng `ClientValidationException` cho lỗi validation

---

### 4. Dependency Injection Configuration

Mỗi Application layer có file `DependencyInjection.cs`:

```csharp
namespace Basket.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        // Exception Handler
        services.AddExceptionHandler<CustomExceptionHandler>();
        
        // FluentValidation - tự động scan assembly
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        
        // MediatR với Pipeline Behaviors
        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            config.AddOpenBehavior(typeof(ValidationBehavior<,>));
            config.AddOpenBehavior(typeof(LoggingBehavior<,>));
        });
        
        return services;
    }
}
```

#### Tại sao mỗi Application Layer cần DI Configuration riêng?

| Lý do | Giải thích |
|-------|-----------|
| **Assembly Scanning** | MediatR và FluentValidation cần scan đúng assembly của từng service để tìm handlers và validators |
| **Dependencies khác nhau** | Mỗi service có repository interfaces, external services riêng (gRPC, HTTP clients) |
| **Database khác nhau** | Basket (Redis), Catalog (PostgreSQL+Marten), Discount (MongoDB), Order (SQL Server) - mỗi service cần register provider riêng |
| **Pipeline Behaviors** | Có thể tùy chỉnh behaviors per service (ví dụ: TransactionBehavior chỉ cho service cần transaction) |
| **Microservices** | Mỗi service deploy độc lập, team khác nhau phát triển, không nên gộp DI configuration |

**Ví dụ thực tế:**
```csharp
// Discount.Application - MongoDB, không cần UnitOfWork
services.AddScoped<ICouponRepository, CouponRepository>();

// Order.Application - SQL Server, cần UnitOfWork  
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IUnitOfWork, UnitOfWork>();

// Basket.Application - Redis
services.AddScoped<IBasketRepository, RedisBasketRepository>();
```

---

#### 4.1. Pipeline Behaviors

**ValidationBehavior.cs:**
```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : class
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        var failures = validationResults
            .Where(r => r.Errors.Count > 0)
            .SelectMany(r => r.Errors)
            .ToList();

        if (failures.Count > 0)
            throw new ClientValidationException(failures);

        return await next();
    }
}
```

**LoggingBehavior.cs:**
```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : class
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        logger.LogInformation("Handling {RequestName}", typeof(TRequest).Name);
        
        var response = await next();
        
        logger.LogInformation("Handled {RequestName}", typeof(TRequest).Name);
        
        return response;
    }
}
```

---

### 5. Repository Pattern

Repository Interfaces được định nghĩa trong Application Layer, Implementation trong Infrastructure Layer.

#### 5.1. Danh sách Repository theo Service

**Basket Service:**
```csharp
public interface IBasketRepository
{
    Task<ShoppingCart?> GetBasketAsync(string userId, CancellationToken cancellationToken = default);
    Task<bool> DeleteBasketAsync(string userId, CancellationToken cancellationToken = default);
    Task<ShoppingCart> StoreBasketAsync(ShoppingCart basket, CancellationToken cancellationToken = default);
}
```

**Inventory Service:**
```csharp
public interface IInventoryItemRepository : IRepository<InventoryItemEntity>
{
    Task<InventoryItemEntity?> GetBySkuAsync(string sku, CancellationToken cancellationToken = default);
    Task<bool> ExistsBySkuAsync(string sku, CancellationToken cancellationToken = default);
}

public interface IInventoryReservationRepository : IRepository<InventoryReservationEntity>
{
    Task<IReadOnlyList<InventoryReservationEntity>> GetByReferenceIdAsync(Guid referenceId, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<InventoryReservationEntity>> GetExpiredReservationsAsync(DateTimeOffset now, CancellationToken cancellationToken = default);
}
```

**Order Service:**
```csharp
public interface IOrderRepository : IRepository<OrderEntity>
{
    Task<OrderEntity?> GetByOrderNoAsync(string orderNo, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<OrderEntity>> GetByCustomerIdAsync(Guid customerId, CancellationToken cancellationToken = default);
}
```

#### 5.2. Base Repository Interface

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
}
```

#### 5.3. Quy tắc Repository

- Interface định nghĩa trong `Application/Repositories/` hoặc `Domain/Repositories/`
- Implementation trong `Infrastructure/Repositories/`
- **KHÔNG** dùng `IApplicationDbContext` trực tiếp trong Application layer
- Sử dụng `IUnitOfWork` để truy cập repositories
- Repository methods có `Include` phải có suffix `WithRelationship`

---

### 6. Unit of Work Pattern

#### 6.1. Interface Definition

**File:** `Services/Inventory/Core/Inventory.Domain/Abstractions/IUnitOfWork.cs`

```csharp
public interface IUnitOfWork : IDisposable, IAsyncDisposable
{
    #region Properties
    
    IInventoryItemRepository InventoryItems { get; }
    IInventoryHistoryRepository InventoryHistories { get; }
    IInventoryReservationRepository InventoryReservations { get; }
    ILocationRepository Locations { get; }
    IInboxMessageRepository InboxMessages { get; }
    IOutboxMessageRepository OutboxMessages { get; }
    bool HasActiveTransaction { get; }
    
    #endregion

    #region Methods
    
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task BeginTransactionAsync(CancellationToken cancellationToken = default);
    Task CommitTransactionAsync(CancellationToken cancellationToken = default);
    Task RollbackTransactionAsync(CancellationToken cancellationToken = default);
    
    #endregion
}
```

#### 6.2. Implementation

**File:** `Services/Inventory/Core/Inventory.Infrastructure/UnitOfWork/UnitOfWork.cs`

```csharp
public class UnitOfWork(
    IInventoryReservationRepository inventoryReservations,
    IInventoryItemRepository inventoryItems,
    IInventoryHistoryRepository inventoryHistories,
    ILocationRepository locations,
    IInboxMessageRepository inboxMessages,
    IOutboxMessageRepository outboxMessages,
    ApplicationDbContext context) : IUnitOfWork
{
    public IInventoryReservationRepository InventoryReservations { get; } = inventoryReservations;
    public IInventoryItemRepository InventoryItems { get; } = inventoryItems;
    public IInventoryHistoryRepository InventoryHistories { get; } = inventoryHistories;
    public ILocationRepository Locations { get; } = locations;
    public IInboxMessageRepository InboxMessages { get; } = inboxMessages;
    public IOutboxMessageRepository OutboxMessages { get; } = outboxMessages;

    private IDbContextTransaction? _currentTransaction;

    public bool HasActiveTransaction => _currentTransaction != null;

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await context.SaveChangesAsync(cancellationToken);
    }

    public async Task BeginTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (_currentTransaction != null)
            throw new InvalidOperationException("A transaction is already in progress.");

        _currentTransaction = await context.Database.BeginTransactionAsync(cancellationToken);
    }

    public async Task CommitTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (_currentTransaction == null)
            throw new InvalidOperationException("No active transaction to commit.");

        try
        {
            await context.SaveChangesAsync(cancellationToken);
            await _currentTransaction.CommitAsync(cancellationToken);
        }
        catch
        {
            await RollbackTransactionAsync(cancellationToken);
            throw;
        }
        finally
        {
            await _currentTransaction.DisposeAsync();
            _currentTransaction = null;
        }
    }

    public async Task RollbackTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (_currentTransaction != null)
        {
            await _currentTransaction.RollbackAsync(cancellationToken);
            await _currentTransaction.DisposeAsync();
            _currentTransaction = null;
        }
    }

    public void Dispose()
    {
        _currentTransaction?.Dispose();
        context.Dispose();
    }

    public async ValueTask DisposeAsync()
    {
        if (_currentTransaction != null)
            await _currentTransaction.DisposeAsync();
        
        await context.DisposeAsync();
    }
}
```

#### 6.3. Sử dụng trong Handler

```csharp
public sealed class CreateOrderCommandHandler(
    IUnitOfWork unitOfWork, 
    ICatalogGrpcService catalogGrpc, 
    IDiscountGrpcService discountGrpc) : ICommandHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        await unitOfWork.BeginTransactionAsync(ct);
        
        try
        {
            // Tạo order
            var order = OrderEntity.Create(command.Dto, command.Actor);
            await unitOfWork.Orders.AddAsync(order, ct);
            
            // Thêm order items
            foreach (var item in command.Dto.Items)
            {
                var orderItem = OrderItemEntity.Create(item);
                order.AddItem(orderItem);
            }
            
            // Save changes và commit
            await unitOfWork.SaveChangesAsync(ct);
            await unitOfWork.CommitTransactionAsync(ct);
            
            return order.Id;
        }
        catch
        {
            await unitOfWork.RollbackTransactionAsync(ct);
            throw;
        }
    }
}
```

---

### 7. Application Services (External Service Interfaces)

Các interfaces để giao tiếp với external services.

#### 7.1. gRPC Services

**ICatalogGrpcService.cs:**
```csharp
namespace Basket.Application.Services;

public interface ICatalogGrpcService
{
    Task<ProductResponse?> GetProductAsync(Guid productId);
    Task<IReadOnlyList<ProductResponse>> GetProductsAsync(IEnumerable<Guid> productIds);
    Task<bool> ValidateProductAsync(Guid productId);
}
```

**Implementation:**
```csharp
namespace Basket.Infrastructure.Services;

public sealed class CatalogGrpcService(
    CatalogProtoService.CatalogProtoServiceClient catalogProto,
    ILogger<CatalogGrpcService> logger) : ICatalogGrpcService
{
    public async Task<ProductResponse?> GetProductAsync(Guid productId)
    {
        try
        {
            var request = new GetProductRequest { ProductId = productId.ToString() };
            var response = await catalogProto.GetProductAsync(request);
            
            return new ProductResponse
            {
                Id = Guid.Parse(response.ProductId),
                Name = response.Name,
                Price = (decimal)response.Price
            };
        }
        catch (RpcException ex)
        {
            logger.LogError(ex, "Error calling Catalog gRPC service");
            throw;
        }
    }
}
```

#### 7.2. External API Services

**ICatalogApiService.cs:**
```csharp
namespace Inventory.Application.Services;

public interface ICatalogApiService
{
    Task<ApiResponse<ProductDto>?> GetProductAsync(string sku);
    Task<ApiResponse<List<ProductDto>>> GetProductsAsync(List<string> skus);
}
```

**Implementation với Refit:**
```csharp
namespace Inventory.Infrastructure.Services;

public sealed class CatalogApiService(
    ICatalogApiClient catalogClient,
    ILogger<CatalogApiService> logger) : ICatalogApiService
{
    public async Task<ApiResponse<ProductDto>?> GetProductAsync(string sku)
    {
        try
        {
            return await catalogClient.GetProductBySkuAsync(sku);
        }
        catch (ApiException ex)
        {
            logger.LogError(ex, "Error calling Catalog API");
            throw;
        }
    }
}
```

#### 7.3. Cloud Services

**IMinIOCloudService.cs:**
```csharp
namespace Catalog.Application.Services;

public interface IMinIOCloudService
{
    Task<string> UploadFileAsync(Stream fileStream, string fileName, string contentType);
    Task<Stream> DownloadFileAsync(string fileName);
    Task DeleteFileAsync(string fileName);
}
```

---

### 8. Exception Handling

Application layer chỉ sử dụng các exceptions sau:

| Exception | HTTP Status | Usage |
|-----------|-------------|-------|
| `ClientValidationException` | 400 | Input validation errors |
| `NotFoundException` | 404 | Resource not found |
| `DomainException` | 400/422 | Domain rule violations |

**KHÔNG** tạo custom exceptions mới hoặc dùng `ArgumentException`, `BadRequestException`.

#### 8.1. Exception Handler

```csharp
public sealed class CustomExceptionHandler(ILogger<CustomExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Exception occurred: {Message}", exception.Message);

        var (statusCode, errorMessage) = exception switch
        {
            ClientValidationException => (StatusCodes.Status400BadRequest, exception.Message),
            NotFoundException => (StatusCodes.Status404NotFound, exception.Message),
            DomainException => (StatusCodes.Status400BadRequest, exception.Message),
            _ => (StatusCodes.Status500InternalServerError, "An unexpected error occurred.")
        };

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(new { error = errorMessage }, cancellationToken);

        return true;
    }
}
```

---

## C. Cấu Trúc Thư Mục Application Layer

```
{Service}.Application/
├── DependencyInjection.cs          # DI configuration
├── ApplicationMarker.cs            # Assembly marker
│
├── Dtos/                           # Data Transfer Objects
│   ├── Products/
│   │   ├── CreateProductDto.cs
│   │   ├── UpdateProductDto.cs
│   │   └── ProductDto.cs
│   └── Categories/
│       ├── CreateCategoryDto.cs
│       └── CategoryDto.cs
│
├── Features/                       # Feature-based organization
│   ├── Products/
│   │   ├── Commands/
│   │   │   ├── CreateProductCommand.cs
│   │   │   ├── CreateProductCommandValidator.cs
│   │   │   ├── UpdateProductCommand.cs
│   │   │   ├── UpdateProductCommandValidator.cs
│   │   │   ├── DeleteProductCommand.cs
│   │   │   └── DeleteProductCommandValidator.cs
│   │   ├── Queries/
│   │   │   ├── GetProductByIdQuery.cs
│   │   │   ├── GetProductByIdQueryValidator.cs
│   │   │   ├── GetAllProductsQuery.cs
│   │   │   └── GetAllProductsQueryValidator.cs
│   │   └── EventHandlers/
│   │       ├── Domain/
│   │       │   └── ProductCreatedDomainEventHandler.cs
│   │       └── Integration/
│   │           └── ProductCreatedIntegrationEventHandler.cs
│   └── Categories/
│       ├── Commands/
│       └── Queries/
│
├── Models/                         # Internal models
│   ├── Responses/
│   │   ├── Externals/              # External service responses
│   │   │   ├── ProductResponse.cs
│   │   │   └── CouponResponse.cs
│   │   └── Internals/              # Internal responses
│   │       └── OrderCalculationResult.cs
│   └── Requests/
│       └── BasketCheckoutRequest.cs
│
├── Repositories/                   # Repository interfaces
│   ├── IProductRepository.cs
│   ├── ICategoryRepository.cs
│   └── IUnitOfWork.cs
│
└── Services/                       # External service interfaces
    ├── ICatalogGrpcService.cs
    ├── IDiscountGrpcService.cs
    └── IMinIOCloudService.cs
```

---

## D. Design Patterns Sử Dụng

### 1. CQRS (Command Query Responsibility Segregation)
- Phân tách Commands (write) và Queries (read)
- Mỗi operation có một class riêng
- Handler xử lý một request duy nhất

### 2. Mediator Pattern
- MediatR làm trung gian giữa sender và receiver
- Giảm coupling giữa components
- Hỗ trợ pipeline behaviors

### 3. Repository Pattern
- Abstract data access logic
- Interface trong Application, Implementation trong Infrastructure
- Dễ dàng test và thay đổi data source

### 4. Unit of Work Pattern
- Quản lý transactions
- Atomic operations
- Rollback khi có lỗi

### 5. Pipeline Behavior Pattern
- Cross-cutting concerns (validation, logging)
- Decorator pattern implementation
- Thực thi tự động trước/sau handler

### 6. Primary Constructor (C# 12)
```csharp
// Cũ
public class Handler : ICommandHandler<Command, Result>
{
    private readonly IRepository _repository;
    
    public Handler(IRepository repository)
    {
        _repository = repository;
    }
}

// Mới - Primary Constructor
public class Handler(IRepository repository) : ICommandHandler<Command, Result>
{
    // repository parameter được tự động tạo field
}
```

### 7. Result Pattern
```csharp
// Query trả về Result object
public sealed record GetProductByIdQuery(Guid ProductId) : IQuery<GetProductByIdResult>;

public sealed record GetProductByIdResult(ProductDto Item);
```

---

## E. Coding Conventions

### 1. File Organization

```csharp
#region using

using System;
using MediatR;
using AutoMapper;

#endregion

namespace Catalog.Application.Features.Products.Commands;

public sealed class CreateProductCommand : ICommand<Guid>
{
    // Implementation
}

#region Validators

public sealed class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    // Implementation
}

#endregion

#region Handlers

public sealed class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, Guid>
{
    #region Fields, Properties and Indexers
    
    private readonly IDocumentSession _session;
    private readonly IMapper _mapper;
    
    #endregion

    #region Ctors
    
    public CreateProductCommandHandler(IDocumentSession session, IMapper mapper)
    {
        _session = session;
        _mapper = mapper;
    }
    
    #endregion

    #region Implementations
    
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken ct)
    {
        // Implementation
    }
    
    #endregion
}

#endregion
```

### 2. Naming Conventions

| Component | Pattern | Example |
|-----------|---------|---------|
| Command | `{Action}{Entity}Command` | `CreateProductCommand` |
| Query | `Get{Entity}Query` | `GetProductByIdQuery` |
| Handler | `{Command/Query}Handler` | `CreateProductCommandHandler` |
| Validator | `{Command}Validator` | `CreateProductCommandValidator` |
| DTO | `{Action}{Entity}Dto` | `CreateProductDto` |
| Result | `{Query}Result` | `GetProductByIdResult` |
| Repository | `I{Entity}Repository` | `IProductRepository` |

### 3. Best Practices

- **Using statements**: Inside `#region using`, sorted alphabetically
- **Read-only queries**: Sử dụng `AsNoTracking()`
- **Repository methods with Include**: Có suffix `WithRelationship`
- **Error messages**: Dùng `MessageCode` constants, không hard-code
- **Primary constructor**: Ưu tiên cho handlers và services
- **Async/await**: Luôn dùng `CancellationToken` parameter

---

## F. Ví dụ Hoàn Chỉnh: Feature Product

### 1. CreateProductCommand.cs

```csharp
#region using

using Catalog.Application.Dtos.Products;
using Catalog.Domain.Entities;
using Catalog.Domain.Enums;
using Marten;
using MediatR;

#endregion

namespace Catalog.Application.Features.Product.Commands;

#region Command

public record CreateProductCommand(CreateProductDto Dto, Actor Actor) : ICommand<Guid>;

#endregion

#region Validator

public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Dto.Name).NotEmpty().WithMessage(MessageCode.ProductNameIsRequired);
        RuleFor(x => x.Dto.Sku).NotEmpty().WithMessage(MessageCode.SkuIsRequired);
        RuleFor(x => x.Dto.Price).GreaterThan(0).WithMessage(MessageCode.PriceMustBeGreaterThanZero);
        RuleFor(x => x.Dto.CategoryId).NotEmpty().WithMessage(MessageCode.CategoryIdIsRequired);
    }
}

#endregion

#region Handler

public class CreateProductCommandHandler(IDocumentSession session) : ICommandHandler<CreateProductCommand, Guid>
{
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken ct)
    {
        // Kiểm tra SKU đã tồn tại chưa
        var exists = await session.Query<ProductEntity>()
            .AnyAsync(x => x.Sku == command.Dto.Sku, ct);
            
        if (exists)
            throw new DomainException(MessageCode.SkuAlreadyExists);

        // Tạo product entity
        var product = new ProductEntity
        {
            Name = command.Dto.Name,
            Sku = command.Dto.Sku,
            Price = command.Dto.Price,
            CategoryId = command.Dto.CategoryId,
            Status = ProductStatus.Draft,
            CreatedBy = command.Actor.UserId,
            CreatedAt = DateTimeOffset.UtcNow
        };

        // Lưu vào database
        session.Store(product);
        await session.SaveChangesAsync(ct);

        return product.Id;
    }
}

#endregion
```

### 2. GetProductByIdQuery.cs

```csharp
#region using

using AutoMapper;
using Catalog.Application.Dtos.Products;
using Catalog.Domain.Entities;
using Marten;
using MediatR;

#endregion

namespace Catalog.Application.Features.Product.Queries;

#region Query

public sealed record GetProductByIdQuery(Guid ProductId) : IQuery<GetProductByIdResult>;

public sealed record GetProductByIdResult(ProductDto Item);

#endregion

#region Handler

public sealed class GetProductByIdQueryHandler(
    IDocumentSession session, 
    IMapper mapper) : IQueryHandler<GetProductByIdQuery, GetProductByIdResult>
{
    public async Task<GetProductByIdResult> Handle(GetProductByIdQuery query, CancellationToken ct)
    {
        var product = await session.Query<ProductEntity>()
            .FirstOrDefaultAsync(x => x.Id == query.ProductId, ct);

        if (product == null)
            throw new NotFoundException(MessageCode.ProductNotFound);

        var dto = mapper.Map<ProductDto>(product);
        
        return new GetProductByIdResult(dto);
    }
}

#endregion
```

### 3. DependencyInjection.cs

```csharp
namespace Catalog.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddExceptionHandler<CustomExceptionHandler>();
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            config.AddOpenBehavior(typeof(ValidationBehavior<,>));
            config.AddOpenBehavior(typeof(LoggingBehavior<,>));
        });

        return services;
    }
}
```

---

## G. Kết Luận

Application Layer trong dự án ProgCoder Shop Microservices được thiết kế rất chuẩn chỉnh theo Clean Architecture và CQRS pattern:

**Điểm mạnh:**
- Clear separation of concerns
- Dễ test (unit test, integration test)
- Dễ mở rộng và maintain
- Tận dụng tốt các features của C# 12
- Consistent patterns across services
- Proper exception handling

**Công nghệ sử dụng:**
- .NET 8
- MediatR 12.x
- FluentValidation 11.x
- AutoMapper 13.x
- Marten 7.x (Document DB)
- Entity Framework Core 8.x
- gRPC

Tổng thể, đây là một implementation xuất sắc của Clean Architecture trong .NET microservices.

---

*Document created: February 7, 2026*
*Author: AI Assistant*
*Project: ProgCoder Shop Microservices*
