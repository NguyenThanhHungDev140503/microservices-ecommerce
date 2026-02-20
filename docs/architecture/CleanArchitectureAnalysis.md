# Phân tích Kiến trúc Clean Architecture - ProG Coder Shop Microservices

## Table of Contents
1. [Tổng quan về cấu trúc thư mục](#tổng-quan-về-cấu-trúc-thư-mục)
2. [Phân tích chi tiết từng layer](#phân-tích-chi-tiết-từng-layer)
3. [Đánh giá việc tuân thủ SOLID và Dependency Rule](#đánh-giá-việc-tuân-thủ-solid-và-dependency-rule)
4. [Sơ đồ luồng dữ liệu và dependency flow](#sơ-đồ-lưu-dữ-liệu-và-dependency-flow)
5. [Phân tích Dependency Injection và Inversion of Control](#phân-tích-dependency-injection-và-inversion-of-control)
6. [Đánh giá ưu điểm, nhược điểm và các vi phạm](#đánh-giá-ưu-điểm-nhược-điểm-và-các-vi-phạm)
7. [Đề xuất cải tiến và best practices](#đề-xuất-cải-thiện-và-best-practices)

---

## 1. Tổng quan về cấu trúc thư mục

### 1.1 Cấu trúc tổng thể của dự án

Dự án được tổ chức theo kiến trúc **Microservices** với mỗi service tuân thủ **Clean Architecture**:

```
src/
├── ApiGateway/                    # API Gateway (YARP)
├── Apps/                         # Frontend Applications
│   ├── App.Admin/                # Admin Dashboard (React)
│   └── App.Store/               # E-commerce Store (React)
├── Services/                     # Microservices
│   ├── Basket/                  # Shopping Cart Service
│   ├── Catalog/                 # Product Catalog Service
│   ├── Discount/                # Coupon/Discount Service
│   ├── Inventory/               # Inventory Management
│   ├── Order/                   # Order Processing
│   ├── Report/                  # Reporting Service
│   ├── Search/                  # Elasticsearch Search
│   ├── Notification/            # Notification Service
│   └── Communication/          # Real-time Communication
├── JobOrchestrator/             # Background Job Scheduler
└── Shared/                       # Shared Libraries
    ├── BuildingBlocks/           # Cross-cutting concerns
    ├── Common/                  # Common utilities
    ├── Contracts/               # gRPC Contracts
    └── EventSourcing/          # Integration Events
```

### 1.2 Cấu trúc của một Microservice (Ví dụ: Catalog Service)

```
Catalog/
├── Api/                        # === PRESENTATION LAYER ===
│   ├── Catalog.Api/            # REST API (Minimal APIs)
│   │   ├── Endpoints/         # API Endpoints (Carter/Minimal APIs)
│   │   ├── Mappings/          # API mapping profiles
│   │   ├── Constants/         # Route constants
│   │   └── Models/           # Request/Response models
│   └── Catalog.Grpc/          # gRPC Service
│       ├── Interceptors/       # API Key validation
│       └── Services/          # gRPC implementation
│
└── Core/                       # === DOMAIN, APPLICATION, INFRASTRUCTURE ===
    │
    ├── Catalog.Domain/          # === DOMAIN LAYER ===
    │   ├── Abstractions/       # Core abstractions
    │   │   ├── Entity.cs       # Base entity
    │   │   ├── Aggregate.cs    # Base aggregate
    │   │   ├── IDomainEvent.cs
    │   │   ├── IAuditable.cs
    │   │   └── ICreationAuditable.cs
    │   ├── Entities/          # Domain entities
    │   │   ├── ProductEntity.cs
    │   │   ├── CategoryEntity.cs
    │   │   └── BrandEntity.cs
    │   ├── Enums/             # Domain enums
    │   │   └── ProductStatus.cs
    │   ├── Events/            # Domain events
    │   │   ├── UpsertedProductDomainEvent.cs
    │   │   └── DeletedUnPublishedProductDomainEvent.cs
    │   └── ValueObjects/      # Value objects
    │       └── Money.cs
    │
    ├── Catalog.Application/     # === APPLICATION LAYER ===
    │   ├── DependencyInjection.cs
    │   ├── Dtos/              # Data Transfer Objects
    │   │   ├── Abstractions/
    │   │   │   ├── EntityDto.cs
    │   │   │   ├── AuditableDto.cs
    │   │   │   └── IDtoId.cs
    │   │   ├── Products/
    │   │   ├── Categories/
    │   │   └── Brands/
    │   ├── Features/           # Use Cases (CQRS)
    │   │   ├── Brand/
    │   │   │   ├── Commands/
    │   │   │   │   ├── CreateBrandCommand.cs
    │   │   │   │   ├── UpdateBrandCommand.cs
    │   │   │   │   └── DeleteBrandCommand.cs
    │   │   │   ├── Queries/
    │   │   │   │   └── GetAllBrandsQuery.cs
    │   │   │   └── EventHandlers/
    │   │   │       └── Domain/
    │   │   ├── Category/
    │   │   └── Product/
    │   ├── Mappings/           # AutoMapper profiles
    │   ├── Models/             # Request/Result models
    │   ├── Repositories/       # Repository interfaces
    │   └── Services/           # Application services interfaces
    │
    └── Catalog.Infrastructure/  # === INFRASTRUCTURE LAYER ===
        ├── DependencyInjection.cs
        ├── Data/
        │   ├── InitialData.cs      # Seed data
        │   ├── BrandSeedData.cs
        │   ├── CategorySeedData.cs
        │   └── ProductSeedData.cs
        ├── Repositories/       # Repository implementations
        │   └── OutboxRepository.cs
        └── Services/           # External services
            └── MinIOCloudService.cs
```

---

## 2. Phân tích chi tiết từng layer

### 2.1 Domain Layer

#### 2.1.1 Entities và Aggregates

**Base Entity Pattern** - Mỗi service đều có base entity class:

```csharp
// src/Services/Catalog/Core/Catalog.Domain/Abstractions/Entity.cs
namespace Catalog.Domain.Abstractions;

public abstract class Entity<T> : IEntityId<T>, IAuditable
{
    public T Id { get; set; } = default!;
    public DateTimeOffset CreatedOnUtc { get; set; }
    public string? CreatedBy { get; set; }
    public DateTimeOffset? LastModifiedOnUtc { get; set; }
    public string? LastModifiedBy { get; set; }
}
```

**Aggregate Root Pattern** - Đối với những entity cần quản lý domain events:

```csharp
// src/Services/Catalog/Core/Catalog.Domain/Abstractions/Aggregate.cs
namespace Catalog.Domain.Abstractions;

public abstract class Aggregate<T> : Entity<T>, IAggregate
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

**Ví dụ Entity cụ thể**:

```csharp
// src/Services/Catalog/Core/Catalog.Domain/Entities/CategoryEntity.cs
namespace Catalog.Domain.Entities;

public sealed class CategoryEntity : Entity<Guid>
{
    public string? Name { get; set; }
    public string? Description { get; set; }
    public string? Slug { get; set; }
    public Guid? ParentId { get; set; }

    // Factory method - Encapsulates creation logic
    public static CategoryEntity Create(Guid id,
        string name,
        string description,
        string slug,
        string performedBy,
        Guid? parentId = null)
    {
        return new CategoryEntity()
        {
            Id = id,
            Name = name,
            Description = description,
            Slug = slug,
            ParentId = parentId,
            CreatedBy = performedBy,
            LastModifiedBy = performedBy,
            CreatedOnUtc = DateTimeOffset.UtcNow,
            LastModifiedOnUtc = DateTimeOffset.UtcNow,
        };
    }
}
```

**Aggregate Root với Domain Events**:

```csharp
// src/Services/Order/Core/Order.Domain/Entities/OrderEntity.cs
namespace Order.Domain.Entities;

public sealed class OrderEntity : Aggregate<Guid>
{
    public string OrderNo { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTimeOffset OrderDate { get; private set; }
    public List<OrderItemEntity> Items { get; private set; }

    // Domain logic encapsulated in the aggregate
    public void UpdateStatus(OrderStatus newStatus, string? reason)
    {
        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Cannot update a cancelled order");

        var oldStatus = Status;
        Status = newStatus;

        if (newStatus == OrderStatus.Delivered)
        {
            AddDomainEvent(new OrderDeliveredDomainEvent(this));
        }
    }
}
```

#### 2.1.2 Value Objects

```csharp
// src/Services/Catalog/Core/Catalog.Domain/ValueObjects/Money.cs
namespace Catalog.Domain.ValueObjects;

public sealed record Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; } = "USD";

    public static Money Create(decimal amount, string currency = "USD")
    {
        if (amount < 0)
            throw new DomainException("Amount cannot be negative");

        return new Money { Amount = amount, Currency = currency };
    }
}
```

#### 2.1.3 Domain Events

```csharp
// src/Services/Order/Core/Order.Domain/Events/OrderCreatedDomainEvent.cs
namespace Order.Domain.Events;

public sealed record OrderCreatedDomainEvent(OrderEntity Order) : IDomainEvent;
```

#### 2.1.4 Repository Interfaces (định nghĩa ở Domain Layer)

```csharp
// src/Services/Discount/Core/Discount.Application/Repositories/ICouponRepository.cs
namespace Discount.Application.Repositories;

public interface ICouponRepository
{
    Task<CouponEntity?> GetByIdAsync(Guid id, CancellationToken cancellationToken);
    Task<CouponEntity?> GetByCodeAsync(string code, CancellationToken cancellationToken);
    Task<IReadOnlyList<CouponEntity>> GetAllAsync(CancellationToken cancellationToken);
    Task<Guid> CreateAsync(CouponEntity coupon, CancellationToken cancellationToken);
    Task<bool> UpdateAsync(CouponEntity coupon, CancellationToken cancellationToken);
    Task<bool> DeleteAsync(Guid id, CancellationToken cancellationToken);
}
```

---

### 2.2 Application Layer

#### 2.2.1 CQRS Pattern với MediatR

**Commands** (để write operations):

```csharp
// src/Services/Catalog/Core/Catalog.Application/Features/Product/Commands/CreateProductCommand.cs
namespace Catalog.Application.Features.Product.Commands;

public record CreateProductCommand(CreateProductDto Dto, Actor Actor)
    : ICommand<Guid>;

public class CreateProductCommandHandler
    : ICommandHandler<CreateProductCommand, Guid>
{
    private readonly IDocumentSession _session;
    private readonly IMinIOCloudService _minIO;
    private readonly ISender _sender;
    private readonly IMapper _mapper;

    public CreateProductCommandHandler(
        IDocumentSession session,
        IMinIOCloudService minIO,
        ISender sender)
    {
        _session = session;
        _minIO = minIO;
        _sender = sender;
    }

    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        // 1. Upload images to MinIO
        var imageUrls = await Task.WhenAll(
            request.Dto.Images.Select(img => _minIO.UploadImageAsync(img, cancellationToken))
        );

        // 2. Create entity
        var product = ProductEntity.Create(
            Guid.NewGuid(),
            request.Dto.Name,
            request.Dto.Description,
            request.Dto.Price,
            request.Dto.Currency,
            request.Actor.Name,
            imageUrls
        );

        // 3. Persist
        _session.Store(product);
        await _session.SaveChangesAsync(cancellationToken);

        // 4. Publish domain events
        await _sender.Send(new UpsertedProductDomainEvent(product.Id));

        return product.Id;
    }
}
```

**Queries** (để read operations):

```csharp
// src/Services/Catalog/Core/Catalog.Application/Features/Product/Queries/GetPublishProductsQuery.cs
namespace Catalog.Application.Features.Product.Queries;

public sealed record GetPublishProductsQuery(
    GetPublishProductsFilter Filter,
    PaginationRequest Paging
) : IQuery<GetPublishProductsResult>;

public sealed class GetPublishProductsQueryHandler
    : IQueryHandler<GetPublishProductsQuery, GetPublishProductsResult>
{
    private readonly IDocumentSession _session;
    private readonly IMapper _mapper;

    public GetPublishProductsQueryHandler(IDocumentSession session, IMapper mapper)
    {
        _session = session;
        _mapper = mapper;
    }

    public async Task<GetPublishProductsResult> Handle(
        GetPublishProductsQuery request,
        CancellationToken cancellationToken)
    {
        var query = _session.Query<ProductEntity>()
            .Where(p => p.Status == ProductStatus.Published);

        if (!string.IsNullOrEmpty(request.Filter.Search))
        {
            query = query.Where(p =>
                p.Name.Contains(request.Filter.Search) ||
                p.Description.Contains(request.Filter.Search));
        }

        var total = await query.CountAsync(cancellationToken);
        var products = await query
            .OrderBy(p => p.CreatedOnUtc)
            .Skip((request.Paging.Page - 1) * request.Paging.PageSize)
            .Take(request.Paging.PageSize)
            .ToListAsync(cancellationToken);

        var dtos = _mapper.Map<List<ProductDto>>(products);

        return new GetPublishProductsResult(dtos, total, request.Paging);
    }
}
```

#### 2.2.2 DTOs (Data Transfer Objects)

```csharp
// src/Services/Catalog/Core/Catalog.Application/Dtos/Products/ProductDto.cs
namespace Catalog.Application.Dtos.Products;

public sealed record ProductDto : EntityDto<Guid>
{
    public string? Name { get; init; }
    public string? Description { get; init; }
    public decimal Price { get; init; }
    public string? Currency { get; init; }
    public ProductStatus Status { get; init; }
    public List<string> Images { get; init; } = new();
}

// Abstract base DTO
// src/Services/Catalog/Core/Catalog.Application/Dtos/Abstractions/EntityDto.cs
namespace Catalog.Application.Dtos.Abstractions;

public abstract class EntityDto<T> : IDtoId<T>, IAuditableDto
{
    public T Id { get; init; } = default!;
    public DateTimeOffset CreatedOnUtc { get; init; }
    public string? CreatedBy { get; init; }
    public DateTimeOffset? LastModifiedOnUtc { get; init; }
    public string? LastModifiedBy { get; init; }
}
```

#### 2.2.3 Application Services (cho logic cross-boundary)

```csharp
// src/Services/Discount/Core/Discount.Application/Services/IDiscountGrpcService.cs
namespace Discount.Application.Services;

public interface IDiscountGrpcService
{
    Task<EvaluateCouponResponse> EvaluateCouponAsync(
        string code,
        decimal orderTotal,
        CancellationToken cancellationToken = default);
}
```

#### 2.2.4 Domain Event Handlers (trong Application Layer)

```csharp
// src/Services/Order/Core/Order.Application/Features/Order/EventHandlers/Domain/OrderCreatedDomainEventHandler.cs
namespace Order.Application.Features.Order.EventHandlers.Domain;

public sealed class OrderCreatedDomainEventHandler
    : INotificationHandler<OrderCreatedDomainEvent>
{
    private readonly ILogger<OrderCreatedDomainEventHandler> _logger;

    public OrderCreatedDomainEventHandler(ILogger<OrderCreatedDomainEventHandler> logger)
    {
        _logger = logger;
    }

    public Task Handle(OrderCreatedDomainEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Order created event: {OrderId}", notification.Order.Id);
        // Logic để xử lý khi order được tạo
        return Task.CompletedTask;
    }
}
```

#### 2.2.5 Dependency Injection Setup

```csharp
// src/Services/Catalog/Core/Catalog.Application/DependencyInjection.cs
namespace Catalog.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        // Exception handling
        services.AddExceptionHandler<CustomExceptionHandler>();

        // FluentValidation
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        // MediatR with CQRS and Behaviors
        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            config.AddOpenBehavior(typeof(ValidationBehavior<,>));
            config.AddOpenBehavior(typeof(LoggingBehavior<,>));
        });

        // Feature Management (Azure App Configuration style)
        services.AddFeatureManagement();

        // AutoMapper
        services.AddAutoMapper(Assembly.GetExecutingAssembly());

        return services;
    }
}
```

---

### 2.3 Infrastructure Layer

#### 2.3.1 Repository Implementations

```csharp
// src/Services/Discount/Core/Discount.Infrastructure/Repositories/CouponRepository.cs
namespace Discount.Infrastructure.Repositories;

public class CouponRepository : ICouponRepository
{
    private readonly IMongoCollection<CouponEntity> _collection;

    public CouponRepository(IMongoClient mongoClient, IConfiguration config)
    {
        var database = mongoClient.GetDatabase(config["DatabaseName"]);
        _collection = database.GetCollection<CouponEntity>(MongoCollection.Coupon);
    }

    public async Task<CouponEntity?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await _collection.Find(x => x.Id == id)
            .FirstOrDefaultAsync(cancellationToken);
    }

    public async Task<CouponEntity?> GetByCodeAsync(string code, CancellationToken cancellationToken)
    {
        return await _collection.Find(x => x.Code.Value == code)
            .FirstOrDefaultAsync(cancellationToken);
    }

    public async Task<Guid> CreateAsync(CouponEntity coupon, CancellationToken cancellationToken)
    {
        await _collection.InsertOneAsync(coupon, cancellationToken);
        return coupon.Id;
    }

    public async Task<bool> UpdateAsync(CouponEntity coupon, CancellationToken cancellationToken)
    {
        var result = await _collection.ReplaceOneAsync(
            x => x.Id == coupon.Id,
            coupon,
            cancellationToken: cancellationToken);

        return result.ModifiedCount > 0;
    }

    public async Task<bool> DeleteAsync(Guid id, CancellationToken cancellationToken)
    {
        var result = await _collection.DeleteOneAsync(
            x => x.Id == id,
            cancellationToken);

        return result.DeletedCount > 0;
    }
}
```

**Generic Repository Pattern** (sử dụng Entity Framework):

```csharp
// src/Services/Order/Core/Order.Infrastructure/Repositories/Repository.cs
namespace Order.Infrastructure.Repositories;

public class Repository<T>(ApplicationDbContext context)
    : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext Context = context;
    protected readonly DbSet<T> DbSet = context.Set<T>();

    public virtual async Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await DbSet.FindAsync(new object[] { id }, cancellationToken);
    }

    public virtual async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken)
    {
        return await DbSet.ToListAsync(cancellationToken);
    }

    public virtual async Task<T> AddAsync(T entity, CancellationToken cancellationToken)
    {
        await DbSet.AddAsync(entity, cancellationToken);
        return entity;
    }

    public virtual void Update(T entity)
    {
        DbSet.Update(entity);
    }

    public virtual void Delete(T entity)
    {
        DbSet.Remove(entity);
    }
}
```

**Unit of Work Pattern**:

```csharp
// src/Services/Order/Core/Order.Infrastructure/UnitOfWork/UnitOfWork.cs
namespace Order.Infrastructure.UnitOfWork;

public sealed class UnitOfWork(ApplicationDbContext context) : IUnitOfWork
{
    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Dispatch domain events before saving
        await DispatchDomainEventsAsync();

        return await context.SaveChangesAsync(cancellationToken);
    }

    private async Task DispatchDomainEventsAsync()
    {
        var domainEvents = context.ChangeTracker
            .Entries<AggregateRoot>()
            .SelectMany(x => x.Entity.DomainEvents)
            .ToList();

        foreach (var domainEvent in domainEvents)
        {
            // Publish via MediatR
        }
    }
}
```

#### 2.3.2 Database Interceptors

```csharp
// src/Services/Inventory/Core/Inventory.Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs
namespace Inventory.Infrastructure.Data.Interceptors;

public sealed class DispatchDomainEventsInterceptor
    : SaveChangesInterceptor
{
    private readonly IPublisher _publisher;

    public DispatchDomainEventsInterceptor(IPublisher publisher)
    {
        _publisher = publisher;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var dbContext = eventData.Context;
        if (dbContext == null) return base.SavingChangesAsync(eventData, result, cancellationToken);

        // Publish all domain events before saving
        var entries = dbContext.ChangeTracker
            .Entries<IAggregate>()
            .Where(e => e.Entity.DomainEvents.Any())
            .ToList();

        var domainEvents = entries
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();

        entries.ForEach(e => e.Entity.ClearDomainEvents());

        foreach (var domainEvent in domainEvents)
        {
            await _publisher.Publish(domainEvent, cancellationToken);
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

```csharp
// src/Services/Inventory/Core/Inventory.Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs
namespace Inventory.Infrastructure.Data.Interceptors;

public sealed class AuditableEntityInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var dbContext = eventData.Context;
        if (dbContext == null) return base.SavingChangesAsync(eventData, result, cancellationToken);

        var entries = dbContext.ChangeTracker.Entries<IAuditable>()
            .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified);

        var currentTime = DateTimeOffset.UtcNow;
        var currentUser = "System"; // Should come from UserContext

        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedOnUtc = currentTime;
                entry.Entity.CreatedBy = currentUser;
            }
            else
            {
                entry.Entity.LastModifiedOnUtc = currentTime;
                entry.Entity.LastModifiedBy = currentUser;
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

#### 2.3.3 External Services (Infrastructure)

```csharp
// src/Services/Catalog/Core/Catalog.Infrastructure/Services/MinIOCloudService.cs
namespace Catalog.Infrastructure.Services;

public class MinIOCloudService : IMinIOCloudService
{
    private readonly IMinioClient _minioClient;
    private readonly string _bucketName;
    private readonly ILogger<MinIOCloudService> _logger;

    public MinIOCloudService(IMinioClient minioClient, IConfiguration config)
    {
        _minioClient = minioClient;
        _bucketName = config["MinIO:BucketName"]!;
        _logger = logger;
    }

    public async Task<string> UploadImageAsync(byte[] imageData, CancellationToken cancellationToken)
    {
        var fileName = $"{Guid.NewGuid()}.jpg";

        var putObjectArgs = new PutObjectArgs()
            .WithBucket(_bucketName)
            .WithObject(fileName)
            .WithStreamData(new MemoryStream(imageData))
            .WithObjectSize(imageData.Length)
            .WithContentType("image/jpeg");

        await _minioClient.PutObjectAsync(putObjectArgs, cancellationToken);

        var fileUrl = $"{_bucketName}/{fileName}";
        _logger.LogInformation("Uploaded image to MinIO: {FileName}", fileName);

        return fileUrl;
    }

    public async Task<byte[]> DownloadImageAsync(string objectName, CancellationToken cancellationToken)
    {
        var getObjectArgs = new GetObjectArgs()
            .WithBucket(_bucketName)
            .WithObject(objectName);

        var response = await _minioClient.GetObjectAsync(getObjectArgs, cancellationToken);

        using var memoryStream = new MemoryStream();
        await response.CopyToAsync(memoryStream, cancellationToken);

        return memoryStream.ToArray();
    }
}
```

#### 2.3.4 Dependency Injection Setup

```csharp
// src/Services/Report/Core/Report.Infrastructure/DependencyInjection.cs
namespace Report.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration cfg)
    {
        // Auto-register all repositories, services, notification senders
        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => c.Where(t => t.Name.EndsWith("NotificationSender")))
            .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
            .AsImplementedInterfaces()
            .WithTransientLifetime());

        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => c.Where(t => t.Name.EndsWith("Service")))
            .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => t.Name.EndsWith("Repository"))
            .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
            .AsImplementedInterfaces()
            .WithSingletonLifetime());

        // DbContext (PostgreSQL)
        var conn = cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.Database}"];
        var dbName = cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.DatabaseName}"];

        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseNpgsql(conn, opt => opt.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));

        services.AddSingleton<IMongoClient>(sp =>
        {
            var settings = MongoClientSettings.FromConnectionString(conn);
            return new MongoClient(settings);
        });

        return services;
    }
}
```

---

### 2.4 Presentation Layer

#### 2.4.1 Minimal API Endpoints

```csharp
// src/Services/Catalog/Api/Catalog.Api/Endpoints/Products/CreateProduct.cs
namespace Catalog.Api.Endpoints.Products;

public sealed class CreateProduct : CarterModule
{
    private const string Route = CatalogApiRoutes.Products.Base;

    public CreateProduct() : base()
    {
        RequiresAuthorization();
    }

    public override void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost(Route, HandleAsync)
            .WithName(nameof(CreateProduct))
            .WithTags("Products")
            .Produces<Guid>(StatusCodes.Status201Created)
            .Produces<ErrorResult>(StatusCodes.Status400BadRequest)
            .Produces<ErrorResult>(StatusCodes.Status401Unauthorized)
            .Produces<ErrorResult>(StatusCodes.Status500InternalServerError);
    }

    private static async Task<IResult> HandleAsync(
        [FromBody] CreateProductRequest request,
        [FromHeader(Name = ReqHeaderName.Actor)] string actorName,
        ISender sender,
        CancellationToken cancellationToken)
    {
        var command = new CreateProductCommand(
            request.MapToDto(),
            new Actor(actorName)
        );

        var result = await sender.Send(command, cancellationToken);

        return Results.CreatedAtRoute(
            nameof(GetProductById),
            new { productId = result },
            result
        );
    }
}
```

```csharp
// src/Services/Catalog/Api/Catalog.Api/Endpoints/Products/GetPublishProducts.cs
namespace Catalog.Api.Endpoints.Products;

public sealed class GetPublishProducts : CarterModule
{
    private const string Route = $"{CatalogApiRoutes.Products.Base}/published";

    public GetPublishProducts() : base()
    {
        // No authorization required for public products
    }

    public override void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapGet(Route, HandleAsync)
            .WithName(nameof(GetPublishProducts))
            .WithTags("Products")
            .Produces<GetPublishProductsResult>(StatusCodes.Status200OK);
    }

    private static async Task<IResult> HandleAsync(
        [AsParameters] GetPublishProductsFilter filter,
        [AsParameters] PaginationRequest paging,
        ISender sender,
        CancellationToken cancellationToken)
    {
        var query = new GetPublishProductsQuery(filter, paging);
        var result = await sender.Send(query, cancellationToken);

        return Results.Ok(result);
    }
}
```

#### 2.4.2 Program.cs - Entry Point

```csharp
// src/Services/Catalog/Api/Catalog.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Presentation Layer
builder.Services.AddCarter();

// Add Application Layer
builder.Services.AddApplicationServices();

// Add Infrastructure Layer
builder.Services.AddInfrastructureServices(builder.Configuration);

// Add Authentication & Authorization
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options =>
    {
        options.Authority = builder.Configuration["Keycloak:Authority"];
        options.Audience = builder.Configuration["Keycloak:Audience"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            RoleClaimType = ClaimTypes.Role
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy(Policies.AdminOnly, policy =>
        policy.RequireRole(Roles.Admin));
});

// Add Swagger/OpenAPI
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddSwaggerGenExtension();

// Add CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Add Distributed Tracing (OpenTelemetry)
builder.Services.AddDistributedTracing(builder.Configuration);

// Add Serilog
builder.Host.UseSerilog();

var app = builder.Build();

// Middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowAll");
app.UseAuthentication();
app.UseAuthorization();
app.MapCarter();

app.Run();
```

---

## 3. Đánh giá việc tuân thủ SOLID và Dependency Rule

### 3.1 Single Responsibility Principle (SRP)

✅ **Tuân thủ tốt:**

- **Entities**: Mỗi entity chỉ chịu trách nhiệm về chính nó
  - `ProductEntity` chỉ quản lý product data và logic
  - `OrderEntity` chỉ quản lý order logic

- **Command/Query Handlers**: Mỗi handler chỉ làm một việc
  ```csharp
  // CreateProductCommandHandler chỉ handle việc tạo product
  public class CreateProductCommandHandler
      : ICommandHandler<CreateProductCommand, Guid>
  ```

- **Repositories**: Chỉ chịu trách nhiệm truy cập dữ liệu
  ```csharp
  public class CouponRepository : ICouponRepository
  {
      // Chỉ có CRUD operations, không có business logic
  }
  ```

⚠️ **Có thể cải thiện:**
- Một số handlers có nhiều trách nhiệm (upload ảnh, tạo entity, publish event)

### 3.2 Open/Closed Principle (OCP)

✅ **Tuân thủ tốt:**

- **Extension via Interfaces**:
  ```csharp
  public interface INotificationSender
  {
      Task SendAsync(NotificationContext context);
  }

  // Có thể thêm các sender mới mà không sửa code cũ
  public class EmailNotificationSender : INotificationSender { }
  public class DiscordNotificationSender : INotificationSender { }
  public class InAppNotificationSender : INotificationSender { }
  ```

- **Behaviors in MediatR**:
  ```csharp
  // Thêm behavior mới không cần sửa code cũ
  config.AddOpenBehavior(typeof(ValidationBehavior<,>));
  config.AddOpenBehavior(typeof(LoggingBehavior<,>));
  ```

- **Strategy Pattern**:
  ```csharp
  public interface INotificationSenderResolver
  {
      INotificationSender Resolve(ChannelType channel);
  }
  ```

### 3.3 Liskov Substitution Principle (LSP)

✅ **Tuân thủ tốt:**

- **Base Entity/Aggregate**:
  ```csharp
  public abstract class Entity<T> : IEntityId<T>, IAuditable
  // Tất cả concrete entities đều có thể thay thế Entity<T>
  ```

- **Repository Interfaces**:
  ```csharp
  public interface IRepository<T> where T : class
  // Tất cả repository implementations đều có thể thay thế nhau
  ```

⚠️ **Cần kiểm tra:**
- Các implementations của `INotificationSender` có tuân thủ LSP không?

### 3.4 Interface Segregation Principle (ISP)

✅ **Tuân thủ tốt:**

- **Separate interfaces for Commands and Queries**:
  ```csharp
  public interface ICommandNotificationRepository
  {
      Task AddAsync(NotificationEntity entity);
      Task UpdateAsync(NotificationEntity entity);
  }

  public interface IQueryNotificationRepository
  {
      Task<IReadOnlyList<NotificationEntity>> GetAllAsync();
      Task<NotificationEntity?> GetByIdAsync(Guid id);
  }
  ```

- **CQRS Interfaces**:
  ```csharp
  public interface ICommand<TResponse> : IRequest<TResponse> { }
  public interface IQuery<TResponse> : IRequest<TResponse> { }

  public interface ICommandHandler<in TCommand, TResponse> { }
  public interface IQueryHandler<in TQuery, TResponse> { }
  ```

⚠️ **Có thể cải thiện:**
- Một số repository interfaces có quá nhiều methods

### 3.5 Dependency Inversion Principle (DIP)

✅ **Tuân thủ tốt:**

- **High-level modules không phụ thuộc low-level**:
  ```csharp
  // Application Layer (High-level)
  public class CreateProductCommandHandler(
      IMinIOCloudService minIO,   // Interface, not concrete class
      IDocumentSession session,      // Interface
      ISender sender)              // Interface
  {
      // ...
  }
  ```

- **Dependencies injected via DI**:
  ```csharp
  // Infrastructure Layer defines concrete implementations
  services.AddScoped<IMinIOCloudService, MinIOCloudService>();
  ```

- **Domain Layer không phụ thuộc vào bất kỳ layer nào**:
  - Chỉ có abstractions (interfaces, base classes)
  - Không có dependency đến EF Core, MongoDB, etc.

### 3.6 Dependency Rule (Clean Architecture)

```
        ┌─────────────────────────────────────────┐
        │           Presentation Layer            │
        │    (API Endpoints, gRPC Services)    │
        └────────────┬────────────────────────┬──┘
                     │                    │
                     ▼                    ▼
        ┌─────────────────────────────────────────┐
        │         Application Layer             │
        │     (Commands, Queries, DTOs)        │
        └────────────┬────────────────────────┬──┘
                     │                    │
                     ▼                    ▼
        ┌─────────────────────────────────────────┐
        │           Domain Layer                │
        │  (Entities, Value Objects, Events)   │
        └─────────────────────────────────────────┘
                     ▲
                     │
                     │
        ┌────────────┴───────────────────────────┐
        │      Infrastructure Layer             │
        │  (Repositories, DB, External APIs)   │
        └────────────────────────────────────────┘
```

**Dependency Direction:**
- ✅ Presentation → Application (OK)
- ✅ Application → Domain (OK)
- ✅ Infrastructure → Domain (OK)
- ✅ Presentation → Infrastructure (OK - for startup only)
- ✅ Application → Infrastructure (OK - for interfaces only)

**Dependency Violations Check:**

| Layer | Depends On | Valid? | Notes |
|-------|-------------|---------|-------|
| Domain | None | ✅ | Pure abstractions |
| Application | Domain | ✅ | OK |
| Application | Shared.BuildingBlocks | ✅ | Shared abstractions |
| Infrastructure | Domain | ✅ | OK |
| Infrastructure | Shared.Common | ✅ | OK |
| Infrastructure | Application (Interfaces) | ✅ | Only for implementation |
| Presentation | Application | ✅ | OK |
| Presentation | Infrastructure | ⚠️ | Only for DI setup |

---

## 4. Sơ đồ luồng dữ liệu và dependency flow

### 4.1 Request Flow - Create Product

```
┌─────────────┐
│   Client    │ (HTTP POST /api/products)
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Presentation Layer (Catalog.Api)      │
│                                     │
│  CreateProduct Endpoint               │
│  ├─ Validate Actor header            │
│  ├─ Create CreateProductRequest      │
│  └─ Map to CreateProductCommand     │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Application Layer (MediatR)         │
│                                     │
│  ValidationBehavior                  │
│  ├─ FluentValidation                 │
│  └─ Return if invalid               │
│                                     │
│  LoggingBehavior                    │
│  ├─ Log request                    │
│  └─ Log response                   │
│                                     │
│  CreateProductCommandHandler          │
│  ├─ IMinIOCloudService             │
│  │  └─ Upload images to MinIO      │
│  ├─ Create ProductEntity            │
│  ├─ IDocumentSession               │
│  │  └─ Save to PostgreSQL         │
│  └─ Publish Domain Events         │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Domain Layer                        │
│                                     │
│  ProductEntity (Aggregate)           │
│  ├─ Name, Description, Price        │
│  ├─ Status                         │
│  └─ Domain Events                  │
│     └─ UpsertedProductDomainEvent   │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Infrastructure Layer               │
│                                     │
│  MinIOCloudService                  │
│  └─ Upload images to S3-compatible │
│                                     │
│  IDocumentSession (PostgreSQL)       │
│  └─ Execute INSERT                  │
│                                     │
│  DispatchDomainEventsInterceptor      │
│  ├─ Capture events before save       │
│  └─ Publish via MassTransit/RabbitMQ│
└────────────────┬────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │  Response     │ (201 Created)
        │  Product ID   │
        └───────────────┘
```

### 4.2 Query Flow - Get Products

```
┌─────────────┐
│   Client    │ (HTTP GET /api/products/published)
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Presentation Layer (Catalog.Api)      │
│                                     │
│  GetPublishProducts Endpoint          │
│  ├─ Parse filter & pagination       │
│  └─ Create GetPublishProductsQuery   │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Application Layer (MediatR)         │
│                                     │
│  LoggingBehavior                    │
│  ├─ Log request                    │
│                                     │
│  GetPublishProductsQueryHandler       │
│  ├─ IDocumentSession               │
│  ├─ Build LINQ query              │
│  ├─ Apply filters                  │
│  ├─ Apply pagination               │
│  └─ AutoMapper                    │
│     └─ Map Entity → DTO           │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Infrastructure Layer               │
│                                     │
│  IDocumentSession (PostgreSQL)       │
│  └─ Execute SELECT                  │
└────────────────┬────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │  Response     │ (200 OK)
        │  Products[]   │
        │  Total Count  │
        └───────────────┘
```

### 4.3 Domain Event Flow

```
┌─────────────────────────────────────────┐
│  Domain Layer                        │
│                                     │
│  OrderEntity.UpdateStatus()          │
│  └─ AddDomainEvent(                │
│        OrderDeliveredDomainEvent)    │
└────────────────┬────────────────────┘
                 │
                 │ Domain Event Published
                 ▼
┌─────────────────────────────────────────┐
│  Infrastructure Layer               │
│                                     │
│  DispatchDomainEventsInterceptor      │
│  ├─ Capture events before save       │
│  └─ Publish to RabbitMQ            │
└────────────────┬────────────────────┘
                 │
                 │ Integration Event
                 ▼
┌─────────────────────────────────────────┐
│  Event Bus (RabbitMQ/MassTransit)   │
└────────────────┬────────────────────┘
                 │
                 ├──────────────────────────┐
                 ▼                      ▼
┌─────────────────┐         ┌─────────────────┐
│  Notification   │         │  Inventory      │
│  Worker        │         │  Worker         │
│                │         │                 │
│  OrderCreated  │         │  OrderCreated   │
│  EventHandler  │         │  EventHandler  │
│  └─ Send       │         │  └─ Update      │
│     notification│         │     stock        │
└─────────────────┘         └─────────────────┘
```

### 4.4 Cross-Service Communication

```
┌────────────────┐
│  Order API    │
│               │
│  CreateOrder   │
└──────┬─────────┘
       │
       │ gRPC
       ▼
┌────────────────┐
│  Catalog gRPC │◄──────────────────────────┐
│               │                         │
│  GetProduct   │                         │
│  GetPrice     │                         │
└────────────────┘                         │
                                         │ HTTP/gRPC
┌────────────────┐                         │
│  Discount gRPC│──────────────────────────┘
│               │
│  Evaluate     │
│  Coupon       │
└────────────────┘
```

---

## 5. Phân tích Dependency Injection và Inversion of Control

### 5.1 DI Container Configuration

#### Presentation Layer (Program.cs)

```csharp
// src/Services/Catalog/Api/Catalog.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

// 1. Add Presentation Layer (Carter)
builder.Services.AddCarter();

// 2. Add Application Layer (Commands, Queries, DTOs)
builder.Services.AddApplicationServices();

// 3. Add Infrastructure Layer (DB, Repositories, Services)
builder.Services.AddInfrastructureServices(builder.Configuration);

// 4. Add Authentication/Authorization
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(/* ... */);

// 5. Add Swagger/OpenAPI
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddSwaggerGenExtension();

// 6. Add CORS
builder.Services.AddCors(/* ... */);

// 7. Add Distributed Tracing (OpenTelemetry)
builder.Services.AddDistributedTracing(builder.Configuration);

var app = builder.Build();
```

#### Application Layer (AddApplicationServices)

```csharp
// src/Services/Catalog/Core/Catalog.Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        // 1. Exception Handling
        services.AddExceptionHandler<CustomExceptionHandler>();

        // 2. FluentValidation
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        // 3. MediatR with Behaviors
        services.AddMediatR(config =>
        {
            // Register all handlers from this assembly
            config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());

            // Add pipeline behaviors
            config.AddOpenBehavior(typeof(ValidationBehavior<,>));
            config.AddOpenBehavior(typeof(LoggingBehavior<,>));
        });

        // 4. Feature Management
        services.AddFeatureManagement();

        // 5. AutoMapper
        services.AddAutoMapper(Assembly.GetExecutingAssembly());

        return services;
    }
}
```

#### Infrastructure Layer (AddInfrastructureServices)

```csharp
// src/Services/Report/Core/Report.Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration cfg)
    {
        // 1. Auto-register Notification Senders (Strategy Pattern)
        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => c.Where(t => t.Name.EndsWith("NotificationSender")))
            .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
            .AsImplementedInterfaces()
            .WithTransientLifetime());

        // 2. Auto-register Services (gRPC, External APIs)
        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => c.Where(t => t.Name.EndsWith("Service")))
            .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        // 3. Auto-register Repositories
        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
            .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
            .AsImplementedInterfaces()
            .WithSingletonLifetime());

        // 4. Configure DbContext (PostgreSQL)
        var conn = cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.Database}"];
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseNpgsql(conn));

        // 5. Configure MongoDB
        services.AddSingleton<IMongoClient>(sp =>
        {
            var settings = MongoClientSettings.FromConnectionString(conn);
            return new MongoClient(settings);
        });

        return services;
    }
}
```

### 5.2 Service Lifetimes

| Service Type | Lifetime | Example | Rationale |
|--------------|----------|----------|-----------|
| **DbContext** | Scoped | `ApplicationDbContext` | One per HTTP request |
| **Repositories** | Singleton | `CouponRepository` | Stateless, thread-safe |
| **gRPC Clients** | Singleton | `CatalogGrpcClient` | Expensive to create |
| **External Services** | Scoped | `MinIOCloudService` | May have client state |
| **Command Handlers** | Transient (MediatR) | `CreateProductCommandHandler` | Created per request |
| **Query Handlers** | Transient (MediatR) | `GetProductsQueryHandler` | Created per request |
| **Notification Senders** | Transient | `EmailNotificationSender` | Strategy pattern |
| **Application Services** | Scoped | `ICatalogGrpcService` | Per-request context |

### 5.3 Constructor Injection Examples

#### Command Handler

```csharp
// src/Services/Order/Core/Order.Application/Features/Order/Commands/CreateOrderCommandHandler.cs
public sealed class CreateOrderCommandHandler
    : ICommandHandler<CreateOrderCommand, Guid>
{
    // All dependencies injected via constructor
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICatalogGrpcService _catalogGrpc;
    private readonly IDiscountGrpcService _discountGrpc;
    private readonly IMapper _mapper;

    public CreateOrderCommandHandler(
        IUnitOfWork unitOfWork,              // Infrastructure implementation
        ICatalogGrpcService catalogGrpc,   // Infrastructure implementation
        IDiscountGrpcService discountGrpc,  // Infrastructure implementation
        IMapper mapper)                      // AutoMapper
    {
        _unitOfWork = unitOfWork;
        _catalogGrpc = catalogGrpc;
        _discountGrpc = discountGrpc;
        _mapper = mapper;
    }

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        // Use dependencies without knowing concrete implementations
        // ...
    }
}
```

#### Repository Implementation

```csharp
// src/Services/Discount/Core/Discount.Infrastructure/Repositories/CouponRepository.cs
public class CouponRepository : ICouponRepository
{
    private readonly IMongoCollection<CouponEntity> _collection;
    private readonly ILogger<CouponRepository> _logger;

    // Constructor injection
    public CouponRepository(IMongoClient mongoClient, IConfiguration config, ILogger<CouponRepository> logger)
    {
        _logger = logger;

        var database = mongoClient.GetDatabase(config["DatabaseName"]);
        _collection = database.GetCollection<CouponEntity>(MongoCollection.Coupon);
    }

    // ... implementation
}
```

#### API Endpoint

```csharp
// src/Services/Catalog/Api/Catalog.Api/Endpoints/Products/CreateProduct.cs
private static async Task<IResult> HandleAsync(
    [FromBody] CreateProductRequest request,
    [FromHeader(Name = ReqHeaderName.Actor)] string actorName,
    ISender sender,  // MediatR ISender - injected
    CancellationToken cancellationToken)
{
    // Use sender to dispatch command
    var command = new CreateProductCommand(
        request.MapToDto(),
        new Actor(actorName)
    );

    var result = await sender.Send(command, cancellationToken);

    return Results.CreatedAtRoute(
        nameof(GetProductById),
        new { productId = result },
        result
    );
}
```

---

## 6. Đánh giá ưu điểm, nhược điểm và các vi phạm Clean Architecture

### 6.1 Ưu điểm ✅

#### 6.1.1 Tách biệt rõ ràng giữa các layer

- **Domain Layer** thuần túy, không có dependency bên ngoài
- **Application Layer** tập trung vào use cases (CQRS)
- **Infrastructure Layer** đóng gói các implementation details
- **Presentation Layer** chỉ handle HTTP/gRPC concerns

#### 6.1.2 CQRS Pattern được áp dụng tốt

```csharp
// Commands (Write)
public record CreateProductCommand(...) : ICommand<Guid>;

// Queries (Read)
public record GetPublishProductsQuery(...) : IQuery<GetPublishProductsResult>;

// Separated handlers
public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, Guid> { }
public class GetPublishProductsQueryHandler : IQueryHandler<GetPublishProductsQuery, GetPublishProductsResult> { }
```

#### 6.1.3 Domain-Driven Design (DDD) được áp dụng

- **Aggregates**: `OrderEntity`, `ProductEntity` với invariants
- **Value Objects**: `Money`, `Address`, `Customer`
- **Domain Events**: `OrderCreatedDomainEvent`, `StockChangedDomainEvent`
- **Repositories**: Interface định nghĩa ở Domain, implementation ở Infrastructure

#### 6.1.4 Dependency Injection tốt

- Sử dụng DI container (Microsoft.Extensions.DependencyInjection)
- Constructor injection cho tất cả dependencies
- Scrutor cho auto-registration (DRY)
- Service lifetimes được quy định rõ ràng

#### 6.1.5 Cross-Cutting Concerns được tách riêng

```csharp
// src/Shared/BuildingBlocks/
├── Behaviors/
│   ├── LoggingBehavior.cs        // Logging cho tất cả commands/queries
│   └── ValidationBehavior.cs    // Validation cho tất cả requests
├── Exceptions/
│   └── CustomExceptionHandler.cs // Global exception handling
├── CQRS/
│   ├── ICommand.cs
│   ├── IQuery.cs
│   ├── ICommandHandler.cs
│   └── IQueryHandler.cs
└── Pagination/
    └── PaginatedResult.cs
```

#### 6.1.6 Microservices độc lập

- Mỗi service có database riêng (PostgreSQL, MongoDB, Elasticsearch)
- Communication qua gRPC/REST API
- Async event-driven với RabbitMQ/MassTransit
- Scalable độc lập

#### 6.1.7 Testing-friendly

- Interfaces cho tất cả dependencies
- Mock-friendly với DI
- Unit tests cho command/query handlers dễ viết
- Integration tests có thể test từng service riêng

#### 6.1.8 DRY (Don't Repeat Yourself)

- **Shared BuildingBlocks**: CQRS, Behaviors, Exceptions
- **Shared Common**: Utilities, Constants, Configurations
- **Base classes**: `Entity<T>`, `Aggregate<T>`, `EntityDto<T>`

### 6.2 Nhược điểm ⚠️

#### 6.2.1 Số lượng dự án/phân vùng (project) quá nhiều

```
src/Services/Catalog/
├── Api/Catalog.Api.csproj
├── Api/Catalog.Grpc.csproj
├── Core/Catalog.Application.csproj
├── Core/Catalog.Domain.csproj
├── Core/Catalog.Infrastructure.csproj
├── Worker/Catalog.Worker.Outbox.csproj
└── Worker/Catalog.Worker.Consumer.csproj
```

**Vấn đề:**
- Quá nhiều `.csproj` files (6-7 per service)
- Build time lâu
- Deployment phức tạp

**Giải pháp đề xuất:**
```csharp
// Giảm xuống 2-3 projects:
// 1. Catalog.Api (Application + Presentation)
// 2. Catalog.Domain (Domain)
// 3. Catalog.Infrastructure (Infrastructure + Workers)
```

#### 6.2.2 Domain Events chưa được xử lý tối ưu

**Hiện tại:**
```csharp
// Domain events được publish trực tiếp từ Infrastructure layer
// DispatchDomainEventsInterceptor catches events before SaveChanges
```

**Vấn đề:**
- Gắn chặt với EF Core/MongoDB interceptors
- Khó test
- Không có mechanism cho eventual consistency tracking

**Giải pháp đề xuất:**
```csharp
// Dùng Outbox Pattern mạnh mẽ hơn
public interface IDomainEventDispatcher
{
    Task DispatchAsync<T>(T domainEvent) where T : IDomainEvent;
}

// Background service xử lý outbox
public class OutboxBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Đọc từ Outbox table
        // Publish đến message bus
        // Xóa hoặc mark as processed
    }
}
```

#### 6.2.3 AutoMapper profiles ở Application Layer

**Vấn đề:**
```csharp
// src/Services/Catalog/Core/Catalog.Application/Mappings/CatalogMappingProfile.cs
public class CatalogMappingProfile : Profile
{
    public CatalogMappingProfile()
    {
        CreateMap<ProductEntity, ProductDto>();
        CreateMap<CreateProductDto, ProductEntity>();
    }
}
```

- Application Layer không nên biết về Entity từ Domain Layer (về lý thuyết)
- Nhưng trong thực tế, đây là acceptable trade-off

**Giải pháp:**
- Để mapping profile ở Application Layer là OK
- Hoặc dùng manual mapping cho critical operations

#### 6.2.4 Shared Code duplication

```csharp
// Mỗi service có phiên bản riêng của:
// - Entity<T>
// - Aggregate<T>
// - IDomainEvent
// - IAuditable
// - IDomainEvent.cs
```

**Vấn đề:**
- Duplicate code giữa services
- Khi fix bug phải sửa nhiều nơi

**Giải pháp:**
```
src/Shared/
└── Domain/
    ├── Abstractions/
    │   ├── Entity.cs
    │   ├── Aggregate.cs
    │   ├── IAuditable.cs
    │   └── IDomainEvent.cs
    └── Common/
        └── ValueObjects/
            ├── Money.cs
            └── Address.cs
```

#### 6.2.5 Lack of Domain Services

**Hiện tại:**
- Business logic chủ yếu ở Aggregates và Command Handlers
- Không có nhiều Domain Services

**Vấn đề:**
- Complex business logic có thể nằm sai chỗ
- Violates Single Responsibility

**Giải pháp:**
```csharp
// Domain Service example
public interface IPricingService
{
    decimal CalculateDiscountedPrice(decimal originalPrice, CouponEntity coupon);
}

public class PricingService : IPricingService
{
    public decimal CalculateDiscountedPrice(decimal originalPrice, CouponEntity coupon)
    {
        // Pricing logic that doesn't belong in Order or Coupon entities
    }
}
```

### 6.3 Vi phạm Clean Architecture 🔴

#### 6.3.1 Application Layer depends on Domain Entities via AutoMapper

```csharp
// Catalog.Application/Mappings/CatalogMappingProfile.cs
public class CatalogMappingProfile : Profile
{
    public CatalogMappingProfile()
    {
        // Application Layer references Domain Entities
        CreateMap<ProductEntity, ProductDto>();  // ❌ Violation
    }
}
```

**Tính nghiêm trọng:** Medium
**Giải pháp:** Đây là acceptable trade-off. Hoặc dùng manual mapping.

#### 6.3.2 Infrastructure Layer có Domain Events handling logic

```csharp
// Inventory.Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs
public class DispatchDomainEventsInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(...)
    {
        // Infrastructure layer dispatches Domain Events
        // ❌ Domain Event logic ở Infrastructure Layer
    }
}
```

**Tính nghiêm trọng:** Medium
**Giải pháp:** Move dispatching logic ra một DomainEventDispatcher service ở Application Layer.

#### 6.3.3 Direct gRPC calls from Command Handlers

```csharp
// Order.Application/Features/Order/Commands/CreateOrderCommandHandler.cs
public class CreateOrderCommandHandler
{
    private readonly ICatalogGrpcService _catalogGrpc;  // ❌ Direct external call
    private readonly IDiscountGrpcService _discountGrpc;

    public async Task<Guid> Handle(CreateOrderCommand request, ...)
    {
        // Direct gRPC call
        var product = await _catalogGrpc.GetProductAsync(request.ProductId);
    }
}
```

**Tính nghiêm trọng:** Low (Acceptable in microservices)
**Giải pháp:**
- Dùng Integration Events thay vì synchronous calls
- Hoặc Circuit Breaker pattern

#### 6.3.4 Entity Framework attributes on Domain Entities

```csharp
// Order.Domain/Entities/OrderEntity.cs
[Table("Orders")]  // ❌ Infrastructure concern in Domain
public class OrderEntity : Aggregate<Guid>
{
    [Key]  // ❌ Infrastructure concern in Domain
    public override Guid Id { get; set; }
}
```

**Tính nghiêm trọng:** High
**Giải pháp:**
```csharp
// Domain Entity (pure)
public class OrderEntity : Aggregate<Guid>
{
    public override Guid Id { get; set; }
}

// EF Configuration (Infrastructure)
public class OrderEntityConfiguration : IEntityTypeConfiguration<OrderEntity>
{
    public void Configure(EntityTypeBuilder<OrderEntity> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(x => x.Id);
    }
}
```

---

## 7. Đề xuất cải tiến và best practices

### 7.1 Refactor Cấu trúc Dự án

#### 7.1.1 Giảm số lượng projects per service

**Trước:**
```
Catalog/
├── Catalog.Api.csproj
├── Catalog.Grpc.csproj
├── Catalog.Application.csproj
├── Catalog.Domain.csproj
├── Catalog.Infrastructure.csproj
├── Catalog.Worker.Outbox.csproj
└── Catalog.Worker.Consumer.csproj
```

**Sau:**
```
Catalog/
├── Catalog.Web.csproj              (API + gRPC + Presentation)
├── Catalog.Domain.csproj           (Domain)
└── Catalog.Infrastructure.csproj    (Infrastructure + Workers)
```

**Lợi ích:**
- Giảm build time
- Đơn giản hóa deployment
- Ít file `.csproj` để maintain

#### 7.1.2 Move Shared Domain abstractions to Shared project

```
src/Shared/Domain/
├── Abstractions/
│   ├── Entity.cs
│   ├── Aggregate.cs
│   ├── IDomainEvent.cs
│   ├── IAuditable.cs
│   └── IRepository.cs
└── Common/
    └── ValueObjects/
        ├── Money.cs
        └── Email.cs
```

**Usage:**
```csharp
// Instead of:
namespace Catalog.Domain.Abstractions;

// Use:
namespace Shared.Domain.Abstractions;

// Then reference:
// Catalog.Domain.csproj references Shared.Domain.csproj
```

### 7.2 Improve Domain Events Handling

#### 7.2.1 Implement Robust Outbox Pattern

```csharp
// Shared/Domain/Outbox/OutboxMessage.cs
namespace Shared.Domain.Outbox;

public class OutboxMessage
{
    public Guid Id { get; set; }
    public string EventType { get; set; } = null!;
    public string Payload { get; set; } = null!;
    public DateTime OccurredOnUtc { get; set; }
    public DateTime? ProcessedOnUtc { get; set; }
    public OutboxMessageStatus Status { get; set; }
    public int RetryCount { get; set; }
    public string? Error { get; set; }
}

public enum OutboxMessageStatus
{
    Pending,
    Processing,
    Processed,
    Failed
}
```

```csharp
// Shared/Infrastructure/Outbox/OutboxProcessor.cs
namespace Shared.Infrastructure.Outbox;

public class OutboxProcessor : BackgroundService
{
    private readonly IOutboxRepository _outboxRepo;
    private readonly IPublisher _publisher;
    private readonly ILogger<OutboxProcessor> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var messages = await _outboxRepo.GetPendingMessagesAsync(stoppingToken);

                foreach (var message in messages)
                {
                    await ProcessMessageAsync(message, stoppingToken);
                }

                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing outbox messages");
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }
    }

    private async Task ProcessMessageAsync(OutboxMessage message, CancellationToken cancellationToken)
    {
        var eventType = Type.GetType(message.EventType);
        var domainEvent = JsonSerializer.Deserialize(message.Payload, eventType) as IDomainEvent;

        try
        {
            await _outboxRepo.MarkAsProcessingAsync(message.Id, cancellationToken);
            await _publisher.Publish(domainEvent, cancellationToken);
            await _outboxRepo.MarkAsProcessedAsync(message.Id, cancellationToken);
        }
        catch (Exception ex)
        {
            await _outboxRepo.MarkAsFailedAsync(message.Id, ex.Message, cancellationToken);
            _logger.LogError(ex, "Failed to process message {MessageId}", message.Id);
        }
    }
}
```

### 7.3 Add Domain Services

```csharp
// Order.Domain/Services/IPricingService.cs
namespace Order.Domain.Services;

public interface IPricingService
{
    decimal CalculateTotalPrice(List<OrderItem> items, CouponEntity? coupon);
    decimal CalculateDiscount(decimal originalPrice, CouponEntity coupon);
}

// Order.Domain/Services/PricingService.cs
namespace Order.Domain.Services;

public class PricingService : IPricingService
{
    public decimal CalculateTotalPrice(List<OrderItem> items, CouponEntity? coupon)
    {
        var subtotal = items.Sum(item => item.Price * item.Quantity);

        if (coupon != null)
        {
            var discount = CalculateDiscount(subtotal, coupon);
            return Math.Max(0, subtotal - discount);
        }

        return subtotal;
    }

    public decimal CalculateDiscount(decimal originalPrice, CouponEntity coupon)
    {
        return coupon.Type switch
        {
            CouponType.Percentage => originalPrice * coupon.Value / 100,
            CouponType.FixedAmount => Math.Min(originalPrice, coupon.Value),
            _ => 0
        };
    }
}
```

**Use in Command Handler:**
```csharp
// Order.Application/Features/Order/Commands/CreateOrderCommandHandler.cs
public class CreateOrderCommandHandler
{
    private readonly IPricingService _pricingService;  // Domain Service

    public async Task<Guid> Handle(CreateOrderCommand request, ...)
    {
        // Use domain service
        var totalPrice = _pricingService.CalculateTotalPrice(items, coupon);
    }
}
```

### 7.4 Separate EF Configuration from Domain

```csharp
// Order.Infrastructure/Data/Configurations/OrderEntityConfiguration.cs
public class OrderEntityConfiguration : IEntityTypeConfiguration<OrderEntity>
{
    public void Configure(EntityTypeBuilder<OrderEntity> builder)
    {
        builder.ToTable("Orders");

        builder.HasKey(x => x.Id);

        builder.Property(x => x.OrderNo)
            .IsRequired()
            .HasMaxLength(50);

        builder.OwnsOne(x => x.ShippingAddress);

        builder.HasMany(x => x.Items)
            .WithOne(x => x.Order)
            .HasForeignKey(x => x.OrderId);
    }
}
```

**Pure Domain Entity:**
```csharp
// Order.Domain/Entities/OrderEntity.cs
public class OrderEntity : Aggregate<Guid>
{
    public string OrderNo { get; private set; }
    public Address ShippingAddress { get; private set; }
    public List<OrderItem> Items { get; private set; } = new();
}
```

### 7.5 Implement Circuit Breaker for External Calls

```csharp
// Shared/Infrastructure/Resilience/CircuitBreakerExtensions.cs
namespace Shared.Infrastructure.Resilience;

public static class CircuitBreakerExtensions
{
    public static IHttpClientBuilder AddCircuitBreaker(this IHttpClientBuilder builder)
    {
        return builder.AddResilienceHandler("circuit-breaker", configure =>
        {
            configure.AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 0.1,                // 10% failures trigger circuit
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 5,
                DurationOfBreak = TimeSpan.FromSeconds(30),
                ShouldHandle = ex => ex is HttpRequestException
            });

            configure.AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromSeconds(1),
                BackoffType = DelayBackoffType.Exponential
            });
        });
    }
}
```

```csharp
// Program.cs
builder.Services.AddHttpClient<ICatalogGrpcClient, CatalogGrpcClient>()
    .AddCircuitBreaker();
```

### 7.6 Add Comprehensive Documentation

#### 7.6.1 API Documentation (Swagger)

```csharp
// src/Shared/BuildingBlocks/Swagger/SwaggerGenExtension.cs
public static class SwaggerGenExtension
{
    public static IServiceCollection AddSwaggerGenExtension(this IServiceCollection services)
    {
        services.AddSwaggerGen(options =>
        {
            options.SwaggerDoc("v1", new OpenApiInfo
            {
                Title = "ProG Coder Shop - Catalog API",
                Version = "v1",
                Description = "Product Catalog Service API",
                Contact = new OpenApiContact
                {
                    Name = "ProG Coder",
                    Email = "support@progcoder.com"
                }
            });

            // Add JWT authentication
            options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
            {
                Description = "JWT Authorization header using the Bearer scheme",
                Name = "Authorization",
                In = ParameterLocation.Header,
                Type = SecuritySchemeType.ApiKey,
                Scheme = "Bearer"
            });

            options.AddSecurityRequirement(new OpenApiSecurityRequirement
            {
                {
                    new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Type = ReferenceType.SecurityScheme,
                            Id = "Bearer"
                        }
                    },
                    Array.Empty<string>()
                }
            });

            // Include XML comments
            var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
            var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
            options.IncludeXmlComments(xmlPath);
        });

        return services;
    }
}
```

#### 7.6.2 Architecture Documentation

```
docs/
├── architecture/
│   ├── overview.md
│   ├── clean-architecture.md
│   ├── cqrs-pattern.md
│   └── microservices.md
├── api/
│   ├── catalog-api.md
│   ├── order-api.md
│   └── inventory-api.md
└── deployment/
    ├── kubernetes.md
    └── docker-compose.md
```

### 7.7 Add Observability

#### 7.7.1 Distributed Tracing

```csharp
// src/Shared/BuildingBlocks/DistributedTracing/DistributedTracingExtension.cs
public static class DistributedTracingExtension
{
    public static IServiceCollection AddDistributedTracing(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var tracingEndpoint = configuration["Tracing:Endpoint"];

        services.AddOpenTelemetry()
            .ConfigureResource(resource => resource
                .AddService("catalog-api"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(tracingEndpoint);
            });

        return services;
    }
}
```

#### 7.7.2 Logging

```csharp
// src/Shared/BuildingBlocks/Logging/SerilogLoggingExtensions.cs
public static class SerilogLoggingExtensions
{
    public static IHostBuilder UseSerilog(this IHostBuilder hostBuilder)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
            .MinimumLevel.Override("System", LogEventLevel.Warning)
            .Enrich.FromLogContext()
            .Enrich.WithProperty("Service", "Catalog")
            .WriteTo.Console(
                outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
            .WriteTo.Seq("http://localhost:5341")
            .WriteTo.Elasticsearch(...)
            .CreateLogger();

        return hostBuilder.UseSerilog();
    }
}
```

### 7.8 Improve Testing

#### 7.8.1 Unit Test Structure

```
tests/
├── Catalog.UnitTests/
│   ├── Domain/
│   │   ├── ProductEntityTests.cs
│   │   └── ValueObjects/
│   │       └── MoneyTests.cs
│   ├── Application/
│   │   ├── Commands/
│   │   │   ├── CreateProductCommandHandlerTests.cs
│   │   │   └── UpdateProductCommandHandlerTests.cs
│   │   └── Queries/
│   │       └── GetPublishProductsQueryHandlerTests.cs
│   └── Infrastructure/
│       └── Repositories/
│           └── ProductRepositoryTests.cs
├── Order.UnitTests/
│   └── ...
└── IntegrationTests/
    ├── CreateOrderIntegrationTests.cs
    └── OrderFlowIntegrationTests.cs
```

#### 7.8.2 Test Fixtures

```csharp
// tests/Shared/Fixtures/CommandHandlerFixture.cs
public class CommandHandlerFixture<THandler> : IAsyncLifetime
{
    protected ServiceProvider ServiceProvider { get; private set; }

    public THandler Handler => ServiceProvider.GetRequiredService<THandler>();

    public async Task InitializeAsync()
    {
        var services = new ServiceCollection();

        // Add in-memory DbContext
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseInMemoryDatabase("TestDb"));

        // Add MediatR
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(THandler).Assembly));

        // Add AutoMapper
        services.AddAutoMapper(typeof(THandler).Assembly);

        // Add mocks
        services.AddScoped<ICatalogGrpcClient, MockCatalogGrpcClient>();
        services.AddScoped<IDiscountGrpcClient, MockDiscountGrpcClient>();

        ServiceProvider = services.BuildServiceProvider();
    }

    public async Task DisposeAsync()
    {
        await ServiceProvider.DisposeAsync();
    }
}
```

```csharp
// tests/Catalog.UnitTests/Application/Commands/CreateProductCommandHandlerTests.cs
public class CreateProductCommandHandlerTests : IClassFixture<CommandHandlerFixture<CreateProductCommandHandler>>
{
    private readonly CommandHandlerFixture<CreateProductCommandHandler> _fixture;

    public CreateProductCommandHandlerTests(CommandHandlerFixture<CreateProductCommandHandler> fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task Handle_ValidRequest_ReturnsProductId()
    {
        // Arrange
        var request = new CreateProductDto
        {
            Name = "Test Product",
            Description = "Test Description",
            Price = 100.00m,
            Currency = "USD"
        };

        var command = new CreateProductCommand(request, new Actor("TestUser"));

        // Act
        var result = await _fixture.Handler.Handle(command, CancellationToken.None);

        // Assert
        Assert.NotEqual(Guid.Empty, result);
    }

    [Theory]
    [InlineData(-1)]
    [InlineData(0)]
    public async Task Handle_InvalidPrice_ThrowsValidationException(decimal price)
    {
        // Arrange
        var request = new CreateProductDto
        {
            Name = "Test Product",
            Price = price
        };

        var command = new CreateProductCommand(request, new Actor("TestUser"));

        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(() =>
            _fixture.Handler.Handle(command, CancellationToken.None));
    }
}
```

### 7.9 Best Practices Summary

#### 7.9.1 Domain Layer

✅ **DO:**
- Keep Domain entities pure (no EF attributes)
- Use Value Objects for concepts without identity
- Encapsulate business logic in Aggregates
- Use Domain Events for cross-aggregate communication
- Define Repository interfaces in Domain

❌ **DON'T:**
- Add EF attributes on Domain entities
- Call external services from Domain
- Have dependencies on Infrastructure
- Violate aggregate invariants

#### 7.9.2 Application Layer

✅ **DO:**
- Use CQRS (Commands/Queries separation)
- Use MediatR for handler dispatch
- Have thin Command/Query handlers
- Use DTOs for data transfer
- Validate at the Application layer (FluentValidation)

❌ **DON'T:**
- Put business logic in DTOs
- Have circular dependencies
- Expose Domain entities directly
- Mix read and write operations

#### 7.9.3 Infrastructure Layer

✅ **DO:**
- Implement Repository interfaces from Domain
- Use EF Core Configuration classes
- Implement external service calls
- Handle cross-cutting concerns (logging, caching)
- Use interceptors for concerns like auditing

❌ **DON'T:**
- Put business logic here
- Expose Infrastructure types to Application layer (except via interfaces)
- Have Infrastructure-specific types in Domain
- Violate Dependency Inversion Principle

#### 7.9.4 Presentation Layer

✅ **DO:**
- Keep thin (just HTTP/gRPC handling)
- Map requests to Commands/Queries
- Return appropriate HTTP status codes
- Use exception handling middleware
- Document APIs (Swagger)

❌ **DON'T:**
- Put business logic here
- Have direct database access
- Expose internal implementation details
- Mix concerns (validation should be in Application)

---

## 8. Conclusion

### 8.1 Tổng quan

ProG Coder Shop Microservices là một dự án **well-architected** với việc áp dụng **Clean Architecture** và **Microservices patterns** một cách có hệ thống.

**Điểm mạnh:**
- ✅ Tách biệt rõ ràng giữa các layers
- ✅ CQRS pattern được áp dụng tốt
- ✅ DDD với Aggregates, Value Objects, Domain Events
- ✅ Dependency Injection mạnh mẽ
- ✅ Cross-cutting concerns được tách riêng

**Điểm cần cải thiện:**
- ⚠️ Số lượng projects quá nhiều
- ⚠️ Duplicate code giữa services
- ⚠️ Domain Events handling có thể cải thiện
- ⚠️ Một số vi phạm Clean Architecture nhỏ (EF attributes in Domain)

### 8.2 Đề xuất ưu tiên

**High Priority:**
1. Giảm số lượng `.csproj` files (consolidate projects)
2. Move shared Domain abstractions to `Shared.Domain` project
3. Remove EF attributes from Domain entities
4. Implement robust Outbox pattern for Domain Events

**Medium Priority:**
5. Add Domain Services for complex business logic
6. Implement Circuit Breaker for external calls
7. Improve observability (tracing, logging)
8. Add comprehensive testing

**Low Priority:**
9. Refactor AutoMapper profiles
10. Add more documentation
11. Improve deployment automation

### 8.3 Đánh giá cuối cùng

**Architecture Score: 8.5/10**

Dự án tuân thủ tốt Clean Architecture principles, có thể scale tốt, maintainability tốt. Với những cải tiến được đề xuất, dự án có thể đạt **9.5/10**.

---

## Appendix: Code Examples Reference

### A.1 Complete Command Handler Example

```csharp
// src/Services/Order/Core/Order.Application/Features/Order/Commands/CreateOrderCommand.cs
namespace Order.Application.Features.Order.Commands;

public sealed record CreateOrderCommand(CreateOrUpdateOrderDto Dto, Actor Actor)
    : ICommand<Guid>;

public sealed class CreateOrderCommandHandler
    : ICommandHandler<CreateOrderCommand, Guid>
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICatalogGrpcService _catalogGrpc;
    private readonly IDiscountGrpcService _discountGrpc;
    private readonly IPricingService _pricingService;  // Domain Service
    private readonly IMapper _mapper;
    private readonly ILogger<CreateOrderCommandHandler> _logger;

    public CreateOrderCommandHandler(
        IUnitOfWork unitOfWork,
        ICatalogGrpcService catalogGrpc,
        IDiscountGrpcService discountGrpc,
        IPricingService pricingService,
        IMapper mapper,
        ILogger<CreateOrderCommandHandler> logger)
    {
        _unitOfWork = unitOfWork;
        _catalogGrpc = catalogGrpc;
        _discountGrpc = discountGrpc;
        _pricingService = pricingService;
        _mapper = mapper;
        _logger = logger;
    }

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        // 1. Validate products exist
        var productIds = request.Dto.Items.Select(i => i.ProductId).Distinct();
        var products = await _catalogGrpc.GetProductsAsync(productIds, cancellationToken);

        if (products.Count != productIds.Count())
        {
            throw new NotFoundException("One or more products not found");
        }

        // 2. Validate coupon if provided
        CouponEntity? coupon = null;
        if (!string.IsNullOrEmpty(request.Dto.CouponCode))
        {
            coupon = await _discountGrpc.GetCouponByCodeAsync(
                request.Dto.CouponCode, cancellationToken);

            if (coupon == null || !coupon.IsActive)
            {
                throw new NotFoundException("Invalid or inactive coupon");
            }
        }

        // 3. Calculate pricing using Domain Service
        var items = request.Dto.Items.Select(dto =>
            new OrderItemEntity
            {
                Id = Guid.NewGuid(),
                ProductId = dto.ProductId,
                ProductName = products.First(p => p.Id == dto.ProductId).Name,
                Quantity = dto.Quantity,
                Price = products.First(p => p.Id == dto.ProductId).Price
            }).ToList();

        var totalPrice = _pricingService.CalculateTotalPrice(items, coupon);
        var discountAmount = coupon != null
            ? _pricingService.CalculateDiscount(totalPrice, coupon)
            : 0;

        // 4. Create Order Aggregate
        var order = OrderEntity.Create(
            orderId: Guid.NewGuid(),
            orderNo: OrderNo.Generate(),
            items: items,
            totalPrice: totalPrice - discountAmount,
            customer: new Customer(request.Dto.CustomerName, request.Dto.CustomerEmail),
            shippingAddress: new Address(
                request.Dto.ShippingAddress.Street,
                request.Dto.ShippingAddress.City,
                request.Dto.ShippingAddress.PostalCode,
                request.Dto.ShippingAddress.Country
            ),
            discount: coupon != null ? new Discount(coupon.Code, discountAmount) : null,
            actor: request.Actor.Name
        );

        // 5. Persist
        await _unitOfWork.OrderRepository.AddAsync(order, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        // 6. Log
        _logger.LogInformation("Order created: {OrderId}, Total: {Total}", order.Id, order.TotalPrice);

        return order.Id;
    }
}
```

### A.2 Complete Query Handler Example

```csharp
// src/Services/Catalog/Core/Catalog.Application/Features/Product/Queries/GetPublishProductsQuery.cs
namespace Catalog.Application.Features.Product.Queries;

public sealed record GetPublishProductsQuery(
    GetPublishProductsFilter Filter,
    PaginationRequest Paging
) : IQuery<GetPublishProductsResult>;

public sealed class GetPublishProductsQueryHandler
    : IQueryHandler<GetPublishProductsQuery, GetPublishProductsResult>
{
    private readonly IDocumentSession _session;
    private readonly IMapper _mapper;
    private readonly ILogger<GetPublishProductsQueryHandler> _logger;

    public GetPublishProductsQueryHandler(
        IDocumentSession session,
        IMapper mapper,
        ILogger<GetPublishProductsQueryHandler> logger)
    {
        _session = session;
        _mapper = mapper;
        _logger = logger;
    }

    public async Task<GetPublishProductsResult> Handle(
        GetPublishProductsQuery request,
        CancellationToken cancellationToken)
    {
        var query = _session.Query<ProductEntity>()
            .Where(p => p.Status == ProductStatus.Published);

        // Apply search filter
        if (!string.IsNullOrEmpty(request.Filter.Search))
        {
            query = query.Where(p =>
                p.Name.Contains(request.Filter.Search) ||
                p.Description.Contains(request.Filter.Search));
        }

        // Apply category filter
        if (request.Filter.CategoryId.HasValue)
        {
            query = query.Where(p => p.CategoryId == request.Filter.CategoryId.Value);
        }

        // Apply price range filter
        if (request.Filter.MinPrice.HasValue)
        {
            query = query.Where(p => p.Price >= request.Filter.MinPrice.Value);
        }

        if (request.Filter.MaxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= request.Filter.MaxPrice.Value);
        }

        // Get total count
        var total = await query.CountAsync(cancellationToken);

        // Apply pagination
        var products = await query
            .OrderBy(p => p.CreatedOnUtc)
            .Skip((request.Paging.Page - 1) * request.Paging.PageSize)
            .Take(request.Paging.PageSize)
            .ToListAsync(cancellationToken);

        // Map to DTOs
        var dtos = _mapper.Map<List<ProductDto>>(products);

        // Log
        _logger.LogInformation("Retrieved {Count} products out of {Total}", dtos.Count, total);

        return new GetPublishProductsResult(dtos, total, request.Paging);
    }
}
```

### A.3 Complete Domain Event Handler Example

```csharp
// src/Services/Order/Core/Order.Application/Features/Order/EventHandlers/Domain/OrderCreatedDomainEventHandler.cs
namespace Order.Application.Features.Order.EventHandlers.Domain;

public sealed class OrderCreatedDomainEventHandler
    : INotificationHandler<OrderCreatedDomainEvent>
{
    private readonly ILogger<OrderCreatedDomainEventHandler> _logger;
    private readonly INotificationHubService _notificationHub;  // SignalR

    public OrderCreatedDomainEventHandler(
        ILogger<OrderCreatedDomainEventHandler> logger,
        INotificationHubService notificationHub)
    {
        _logger = logger;
        _notificationHub = notificationHub;
    }

    public async Task Handle(OrderCreatedDomainEvent notification, CancellationToken cancellationToken)
    {
        var order = notification.Order;

        _logger.LogInformation(
            "Order created: {OrderId}, Customer: {CustomerEmail}, Total: {Total}",
            order.Id,
            order.Customer.Email,
            order.TotalPrice
        );

        // Send notification to customer via SignalR
        var notificationDto = new NotificationDto
        {
            Type = NotificationType.OrderCreated,
            Title = "Order Confirmed",
            Message = $"Your order #{order.OrderNo} has been confirmed.",
            UserId = order.Customer.Email
        };

        await _notificationHub.SendNotificationAsync(notificationDto, cancellationToken);
    }
}
```

---

**Document Version:** 1.0
**Last Updated:** January 19, 2026
**Author:** AI Architecture Analysis
