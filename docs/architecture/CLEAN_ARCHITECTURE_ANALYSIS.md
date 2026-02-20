# Phân Tích Kiến Trúc Clean Architecture - ProG Coder Shop Microservices

> **Tài liệu phân tích chuyên sâu về kiến trúc Clean Architecture được áp dụng trong dự án E-commerce Microservices**
>
> **Version:** 1.0.0
> **Date:** January 19, 2026
> **Author:** AI Analysis

---

## Table of Contents

1. [Tổng Quan Về Cấu Trúc Thư Mục](#1-tổng-quan-về-cấu-trúc-thư-mục)
2. [Phân Tích Chi Tiết Từng Layer](#2-phân-tích-chi-tết-từng-layer)
3. [Đánh Giá Nguyên Tắc SOLID và Dependency Rule](#3-đánh-giá-nguyên-tắc-solid-và-dependency-rule)
4. [Sơ Đồ Luồng Dữ Liệu và Dependency Flow](#4-sơ-đồ-luồng-dữ-liệu-và-dependency-flow)
5. [Phân Tích Dependency Injection và Inversion of Control](#5-phân-tích-dependency-injection-và-inversion-of-control)
6. [Đánh Giá Ưu Điểm, Nhược Điểm và Các Vi Phạm](#6-đánh-giá-ưu-điểm-nhược-điểm-và-các-vi-phạm)
7. [Đề Xuất Cải Tiến và Best Practices](#7-đề-xuất-cải-tiến-và-best-practices)

---

## 1. Tổng Quan Về Cấu Trúc Thư Mục

### 1.1 Cấu Trúc Project Level

```
progcoder-shop-microservices/
├── src/
│   ├── ApiGateway/                    # API Gateway (YARP)
│   ├── Apps/                         # Frontend Applications
│   │   ├── App.Admin/               # React Admin Panel
│   │   └── App.Store/               # React Store Frontend
│   ├── JobOrchestrator/              # Quartz Job Scheduler
│   ├── Services/                     # Microservices
│   │   ├── Basket/                  # Shopping Cart Service
│   │   ├── Catalog/                 # Product Catalog Service
│   │   ├── Communication/            # Real-time Communication
│   │   ├── Discount/                 # Coupon/Discount Service
│   │   ├── Inventory/                # Inventory Management
│   │   ├── Notification/             # Notification Service
│   │   ├── Order/                   # Order Processing
│   │   ├── Report/                  # Reporting & Analytics
│   │   └── Search/                  # Elasticsearch Service
│   └── Shared/                      # Shared Libraries
│       ├── BuildingBlocks/            # Cross-cutting concerns
│       ├── Common/                   # Shared configurations
│       ├── Contracts/                # gRPC Protobuf contracts
│       └── EventSourcing/           # Integration Events
```

### 1.2 Cấu Trúc Từng Service (Clean Architecture Pattern)

Mỗi service đều tuân theo cấu trúc 4-layer Clean Architecture:

```
Services/[ServiceName]/
├── Api/                              # Presentation Layer
│   ├── [ServiceName].Api/           # REST API (Minimal API + Carter)
│   └── [ServiceName].Grpc/          # gRPC Service
├── Core/                              # Core Business Logic
│   ├── [ServiceName].Domain/         # Domain Layer
│   ├── [ServiceName].Application/    # Application Layer
│   └── [ServiceName].Infrastructure/ # Infrastructure Layer
└── Worker/                            # Background Workers
    ├── [ServiceName].Worker.Consumer    # Message Bus Consumers
    └── [ServiceName].Worker.Outbox     # Outbox Pattern Processor
```

### 1.3 Sơ Đồ Kiến Trúc Overall

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Frontend Applications                        │
│                  (App.Admin | App.Store)                        │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │   API Gateway (YARP)  │
                    └───────────┬───────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
         ┌──────────┐    ┌──────────┐   ┌──────────┐
         │ Catalog  │    │  Order   │   │ Basket   │
         │   API    │    │   API    │   │   API    │
         └────┬─────┘    └────┬─────┘   └────┬─────┘
              │                │               │
         ┌────┼────────────────┼───────────────┼─────┐
         │    │                │               │     │
         ▼    ▼                ▼               ▼     ▼
    ┌────────────────────────────────────────────────────┐
    │         Presentation Layer (API + gRPC)         │
    └────────────────────────┬───────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │Application│      │ Domain   │      │Infra     │
    │   Layer   │◄────│  Layer   │─────│  Layer   │
    └──────────┘      └──────────┘      └──────────┘
```

---

## 2. Phân Tích Chi Tiết Từng Layer

### 2.1 Domain Layer (Core Business Logic)

**Path:** `src/Services/[ServiceName]/Core/[ServiceName].Domain/`

#### 2.1.1 Entities

**Mục đích:** Đại diện cho các business entities với identity và lifecycle.

**Ví dụ - ProductEntity:**

```csharp
// src/Services/Catalog/Core/Catalog.Domain/Entities/ProductEntity.cs
namespace Catalog.Domain.Entities;

public sealed class ProductEntity : Aggregate<Guid>
{
    public string? Name { get; set; }
    public string? Sku { get; set; }
    public decimal Price { get; set; }
    public decimal? SalePrice { get; set; }
    public List<Guid>? CategoryIds { get; set; }
    public ProductStatus Status { get; set; }
    public bool Published { get; set; }
    
    // Factory Method
    public static ProductEntity Create(
        Guid id, string name, string sku, string slug,
        decimal price, decimal? salePrice, 
        List<Guid>? categoryIds, Guid? brandId,
        string performedBy)
    {
        var product = new ProductEntity
        {
            Id = id,
            Name = name,
            Sku = sku,
            Status = ProductStatus.OutOfStock,
            Published = false,
            CreatedOnUtc = DateTimeOffset.UtcNow
        };
        return product;
    }
    
    // Domain Behavior
    public void Publish(string performedBy)
    {
        Published = true;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
    }
    
    public void ChangeStatus(ProductStatus status, string performedBy)
    {
        if (Status == status)
        {
            throw new DomainException(MessageCode.DecisionFlowIllegal);
        }
        Status = status;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
    }
}
```

**Đặc điểm nổi bật:**
- Sử dụng `Aggregate<T>` base class để hỗ trợ Domain Events
- Factory methods (`Create`, `Update`) thay vì public constructors
- Encapsulation của business rules (`ChangeStatus` throws exception nếu invalid)
- Tự động tracking audit fields (`CreatedBy`, `ModifiedBy`, timestamps)

#### 2.1.2 Value Objects

**Mục đích:** Đại diện cho các khái niệm không có identity nhưng có ý nghĩa trong domain.

**Ví dụ - Address Value Object:**

```csharp
// src/Services/Order/Core/Order.Domain/ValueObjects/Address.cs
namespace Order.Domain.ValueObjects;

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
    
    public static Address Of(
        string addressLine, string subdivision, string city,
        string country, string stateOrProvince, string postalCode)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(addressLine);
        ArgumentException.ThrowIfNullOrWhiteSpace(subdivision);
        ArgumentException.ThrowIfNullOrWhiteSpace(city);
        ArgumentException.ThrowIfNullOrWhiteSpace(country);
        ArgumentException.ThrowIfNullOrWhiteSpace(stateOrProvince);
        ArgumentException.ThrowIfNullOrWhiteSpace(postalCode);
        
        return new Address(addressLine, subdivision, city, 
                        country, stateOrProvince, postalCode);
    }
}
```

**Đặc điểm:**
- Immutability thông qua private constructor
- Static factory method `Of()` với validation
- Tất cả các properties đều required (no null)

#### 2.1.3 Domain Events

**Mục đích:** Thông báo về các sự kiện quan trọng trong domain để trigger side effects.

**Ví dụ - UpsertedProductDomainEvent:**

```csharp
// src/Services/Catalog/Core/Catalog.Domain/Events/UpsertedProductDomainEvent.cs
namespace Catalog.Domain.Events;

public sealed record UpsertedProductDomainEvent(
    Guid Id,
    string Name,
    string Sku,
    string Slug,
    decimal Price,
    decimal? SalePrice,
    List<string>? Categories,
    List<string>? Images,
    string Thumbnail,
    ProductStatus Status,
    DateTimeOffset CreatedOnUtc,
    string CreatedBy,
    DateTimeOffset? LastModifiedOnUtc,
    string? LastModifiedBy) : IDomainEvent;
```

**Aggregate base class hỗ trợ:**

```csharp
// src/Services/Catalog/Core/Catalog.Domain/Abstractions/Aggregate.cs
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

#### 2.1.4 Repository Interfaces

**Mục đích:** Định nghĩa contracts cho data access mà không phụ thuộc implementation.

**Ví dụ - IOutboxRepository:**

```csharp
// src/Services/Catalog/Core/Catalog.Application/Repositories/IOutboxRepository.cs
namespace Catalog.Application.Repositories;

public interface IOutboxRepository
{
    Task<bool> AddMessageAsync(OutboxMessageEntity message, 
                              CancellationToken cancellationToken = default);
    Task<bool> UpdateMessagesAsync(IEnumerable<OutboxMessageEntity> messages, 
                                CancellationToken cancellationToken = default);
    Task<List<OutboxMessageEntity>> GetAndClaimMessagesAsync(
        int batchSize, CancellationToken cancellationToken = default);
    Task<List<OutboxMessageEntity>> GetAndClaimRetryMessagesAsync(
        int batchSize, CancellationToken cancellationToken = default);
    Task<bool> ReleaseClaimsAsync(IEnumerable<OutboxMessageEntity> messages, 
                                CancellationToken cancellationToken = default);
}
```

**Lưu ý:** Repository interfaces thực sự nằm trong Application layer vì chúng định nghĩa contracts sử dụng bởi Application services.

---

### 2.2 Application Layer (Use Cases & Orchestration)

**Path:** `src/Services/[ServiceName]/Core/[ServiceName].Application/`

#### 2.2.1 CQRS Pattern với MediatR

**Commands (Write Operations):**

```csharp
// src/Services/Catalog/Core/Catalog.Application/Features/Product/Commands/CreateProductCommand.cs
namespace Catalog.Application.Features.Product.Commands;

public record CreateProductCommand(CreateProductDto Dto, Actor Actor) : ICommand<Guid>;

public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Dto)
            .NotNull()
            .WithMessage(MessageCode.BadRequest)
            .DependentRules(() =>
            {
                RuleFor(x => x.Dto.Name)
                    .NotEmpty()
                    .WithMessage(MessageCode.ProductNameIsRequired);
                RuleFor(x => x.Dto.Price)
                    .GreaterThan(1)
                    .WithMessage(MessageCode.PriceIsRequired);
            });
    }
}

public class CreateProductCommandHandler(
    IMapper mapper,
    IDocumentSession session,
    IMinIOCloudService minIO,
    ISender sender) : ICommandHandler<CreateProductCommand, Guid>
{
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken cancellationToken)
    {
        // 1. Begin transaction
        await session.BeginTransactionAsync(cancellationToken);
        
        // 2. Validate
        await ValidateCategoryAsync(command.Dto.CategoryIds, cancellationToken);
        await ValidateBrandAsync(command.Dto.BrandId, cancellationToken);
        
        // 3. Create Entity using Factory Method
        var entity = ProductEntity.Create(
            id: Guid.NewGuid(),
            name: command.Dto.Name!,
            sku: command.Dto.Sku!,
            slug: command.Dto.Name!.Slugify(),
            price: command.Dto.Price,
            salePrice: command.Dto.SalePrice,
            categoryIds: command.Dto.CategoryIds?.Distinct().ToList(),
            brandId: command.Dto.BrandId,
            performedBy: command.Actor.ToString());
        
        // 4. Upload images
        await UploadImagesAsync(command.Dto.UploadImages, entity, cancellationToken);
        await UploadThumbnailAsync(command.Dto.UploadThumbnail, entity, cancellationToken);
        
        // 5. Apply domain behaviors
        entity.UpdateColors(command.Dto.Colors?.Distinct().ToList(), command.Actor.ToString());
        entity.UpdateTags(command.Dto.Tags?.Distinct().ToList(), command.Actor.ToString());
        
        if (command.Dto.Published)
        {
            entity.Publish(command.Actor.ToString());
        }
        
        // 6. Persist
        session.Store(entity);
        await session.SaveChangesAsync(cancellationToken);
        
        // 7. Send internal command if needed
        if (entity.Published)
        {
            await sender.Send(new PublishProductCommand(entity.Id, command.Actor), cancellationToken);
        }
        
        return entity.Id;
    }
}
```

**Queries (Read Operations):**

```csharp
// src/Services/Catalog/Core/Catalog.Application/Features/Product/Queries/GetProductsQuery.cs
namespace Catalog.Application.Features.Product.Queries;

public record GetProductsQuery(GetProductsFilter Filter, Actor Actor) : IQuery<GetProductsResult>;

public class GetProductsQueryHandler(
    IDocumentSession session,
    IMapper mapper) : IQueryHandler<GetProductsQuery, GetProductsResult>
{
    public async Task<GetProductsResult> Handle(GetProductsQuery query, CancellationToken cancellationToken)
    {
        var filter = query.Filter;
        
        var products = await session.Query<ProductEntity>()
            .Where(x => x.Status == ProductStatus.InStock)
            .Skip((filter.PageNumber - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .ToListAsync(cancellationToken);
        
        var totalCount = await session.Query<ProductEntity>()
            .CountAsync(cancellationToken);
        
        return new GetProductsResult
        {
            Items = mapper.Map<List<ProductDto>>(products),
            TotalCount = totalCount,
            PageNumber = filter.PageNumber,
            PageSize = filter.PageSize
        };
    }
}
```

#### 2.2.2 DTOs (Data Transfer Objects)

**Structure:**

```
Dtos/
├── Abstractions/           # Base DTOs
│   ├── AuditableDto.cs
│   ├── DtoId.cs
│   ├── EntityDto.cs
│   ├── ICreationAuditDto.cs
│   └── IModificationAuditDto.cs
├── [EntityName]/
│   ├── Create[Entity]Dto.cs
│   ├── Update[Entity]Dto.cs
│   ├── [Entity]Dto.cs
│   └── [Entity]InfoDto.cs
└── ValueObjects/
    ├── AddressDto.cs
    ├── DiscountDto.cs
    └── ProductDto.cs
```

**Ví dụ:**

```csharp
// src/Services/Catalog/Core/Catalog.Application/Dtos/Products/CreateProductDto.cs
namespace Catalog.Application.Dtos.Products;

public class CreateProductDto
{
    public string? Name { get; set; }
    public string? Sku { get; set; }
    public string? ShortDescription { get; set; }
    public string? LongDescription { get; set; }
    public decimal Price { get; set; }
    public decimal? SalePrice { get; set; }
    public List<Guid>? CategoryIds { get; set; }
    public Guid? BrandId { get; set; }
    public List<string>? Colors { get; set; }
    public List<string>? Sizes { get; set; }
    public List<string>? Tags { get; set; }
    public bool Featured { get; set; }
    public bool Published { get; set; }
    public UploadFileBytes? UploadThumbnail { get; set; }
    public List<UploadFileBytes>? UploadImages { get; set; }
    public string? SEOTitle { get; set; }
    public string? SEODescription { get; set; }
    public string? Unit { get; set; }
    public decimal? Weight { get; set; }
}
```

#### 2.2.3 Application Services

**Mục đích:** Đóng gói các external service calls (gRPC, HTTP, cloud storage).

```csharp
// src/Services/Catalog/Core/Catalog.Application/Services/IMinIOCloudService.cs
namespace Catalog.Application.Services;

public interface IMinIOCloudService
{
    Task<List<UploadFileResult>> UploadFilesAsync(
        List<UploadFileBytes> files, 
        string bucket,
        bool isPublic,
        CancellationToken cancellationToken = default);
    
    Task<byte[]> DownloadFileAsync(
        string bucket,
        string fileName,
        CancellationToken cancellationToken = default);
    
    Task<bool> DeleteFilesAsync(
        List<string> fileNames,
        string bucket,
        CancellationToken cancellationToken = default);
}
```

#### 2.2.4 Domain Event Handlers

```csharp
// src/Services/Catalog/Core/Catalog.Application/Features/Product/EventHandlers/Domain/UpsertedProductDomainEventHandler.cs
namespace Catalog.Application.Features.Product.EventHandlers.Domain;

public class UpsertedProductDomainEventHandler(
    IOutboxRepository outboxRepository,
    ILogger<UpsertedProductDomainEventHandler> logger) 
    : INotificationHandler<UpsertedProductDomainEvent>
{
    public async Task Handle(UpsertedProductDomainEvent domainEvent, 
                          CancellationToken cancellationToken)
    {
        var integrationEvent = new UpsertedProductIntegrationEvent(
            domainEvent.Id,
            domainEvent.Name,
            domainEvent.Sku,
            domainEvent.Slug,
            domainEvent.Price,
            domainEvent.SalePrice,
            domainEvent.Categories,
            domainEvent.Images,
            domainEvent.Thumbnail,
            domainEvent.Status.ToString(),
            domainEvent.CreatedOnUtc,
            domainEvent.CreatedBy);
        
        await outboxRepository.AddMessageAsync(
            new OutboxMessageEntity
            {
                Id = Guid.NewGuid(),
                EventType = nameof(UpsertedProductIntegrationEvent),
                Payload = JsonSerializer.Serialize(integrationEvent),
                OccurredOnUtc = DateTimeOffset.UtcNow
            },
            cancellationToken);
    }
}
```

#### 2.2.5 Dependency Injection Setup

```csharp
// src/Services/Catalog/Core/Catalog.Application/DependencyInjection.cs
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
        services.AddFeatureManagement();
        services.AddAutoMapper(Assembly.GetExecutingAssembly());
        
        return services;
    }
}
```

---

### 2.3 Infrastructure Layer (Technical Details)

**Path:** `src/Services/[ServiceName]/Core/[ServiceName].Infrastructure/`

#### 2.3.1 Database Implementations

**Marten (PostgreSQL) cho Catalog:**

```csharp
// src/Services/Catalog/Core/Catalog.Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services, IConfiguration cfg)
    {
        // Register Marten (PostgreSQL Document Store)
        services.AddMarten(opts =>
        {
            opts.Connection(cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.Database}"]!);
            opts.UseSystemTextJsonForSerialization();
        }).UseLightweightSessions();
        
        // Auto-register Repositories
        services.Scan(s => s
            .FromAssemblyOf<InfrastructureMarker>()
            .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
            .AsImplementedInterfaces()
            .WithScopedLifetime());
        
        // MinIO (S3-compatible object storage)
        services.AddMinio(configureClient => configureClient
            .WithEndpoint(cfg[$"{MinIoCfg.Section}:{MinIoCfg.Endpoint}"])
            .WithCredentials(cfg[$"{MinIoCfg.Section}:{MinIoCfg.AccessKey}"], 
                          cfg[$"{MinIoCfg.Section}:{MinIoCfg.SecretKey}"])
            .Build());
        
        return services;
    }
}
```

**Entity Framework Core cho Order/Inventory:**

```csharp
// src/Services/Order/Core/Order.Infrastructure/Data/ApplicationDbContext.cs
public sealed class ApplicationDbContext : DbContext
{
    public DbSet<OrderEntity> Orders => Set<OrderEntity>();
    public DbSet<OrderItemEntity> OrderItems => Set<OrderItemEntity>();
    public DbSet<OutboxMessageEntity> OutboxMessages => Set<OutboxMessageEntity>();
    public DbSet<InboxMessageEntity> InboxMessages => Set<InboxMessageEntity>();
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(builder);
    }
}
```

**EF Core Interceptors:**

```csharp
// src/Services/Order/Core/Order.Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs
public class DispatchDomainEventsInterceptor(
    IPublisher publisher,
    ILogger<DispatchDomainEventsInterceptor> logger) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, 
        InterceptionResult<int> result, 
        CancellationToken cancellationToken = default)
    {
        var entities = eventData.Context.ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity);
        
        foreach (var entity in entities)
        {
            var domainEvents = entity.ClearDomainEvents();
            foreach (var domainEvent in domainEvents)
            {
                await publisher.Publish(domainEvent, cancellationToken);
                logger.LogInformation("Published domain event: {EventType}", 
                                  domainEvent.GetType().Name);
            }
        }
        
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

#### 2.3.2 Repository Implementations

```csharp
// src/Services/Catalog/Core/Catalog.Infrastructure/Repositories/OutboxRepository.cs
namespace Catalog.Infrastructure.Repositories;

public class OutboxRepository(
    IDocumentSession session, 
    ILogger<OutboxRepository> logger) : IOutboxRepository
{
    public async Task<List<OutboxMessageEntity>> GetAndClaimMessagesAsync(
        int batchSize, CancellationToken cancellationToken = default)
    {
        var now = DateTimeOffset.UtcNow;
        var claimTimeout = TimeSpan.FromMinutes(5);
        var expiredTime = now.Subtract(claimTimeout);
        
        // Query for unprocessed messages
        var messagesToClaim = await session.Query<OutboxMessageEntity>()
            .Where(x => x.ProcessedOnUtc == null
                && (x.ClaimedOnUtc == null || x.ClaimedOnUtc < expiredTime))
            .OrderBy(x => x.OccurredOnUtc)
            .Take(batchSize)
            .ToListAsync(cancellationToken);
        
        if (!messagesToClaim.Any())
        {
            return [];
        }
        
        // Claim messages (optimistic concurrency)
        foreach (var message in messagesToClaim)
        {
            message.Claim(now);
            session.Store(message);
        }
        
        await session.SaveChangesAsync(cancellationToken);
        
        return messagesToClaim.ToList();
    }
}
```

#### 2.3.3 gRPC Clients

```csharp
// src/Services/Order/Core/Order.Infrastructure/GrpcClients/Extensions/GrpcClientExtension.cs
public static class GrpcClientExtension
{
    public static IServiceCollection AddGrpcClients(this IServiceCollection services, IConfiguration cfg)
    {
        var grpcCfg = cfg.GetSection(GrpcClientCfg.Section).Get<GrpcClientCfg>();
        
        services.AddGrpcClient<DiscountGrpc.DiscountGrpcClient>(options =>
        {
            options.Address = new Uri(grpcCfg.DiscountServiceUrl!);
        }).AddInterceptor<GrpcApiKeyInterceptor>();
        
        services.AddGrpcClient<CatalogGrpc.CatalogGrpcClient>(options =>
        {
            options.Address = new Uri(grpcCfg.CatalogServiceUrl!);
        }).AddInterceptor<GrpcApiKeyInterceptor>();
        
        return services;
    }
}

// Interceptor for API Key authentication
public class GrpcApiKeyInterceptor : Interceptor
{
    private readonly GrpcClientCfg _cfg;
    
    public override AsyncUnaryCall<TRequest, TResponse> UnaryAsyncUnary<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var metadata = new Metadata
        {
            { "X-API-Key", _cfg.ApiKey }
        };
        
        var options = context.Options.WithHeaders(metadata);
        context = new ClientInterceptorContext<TRequest, TResponse>(
            context.Method, 
            context.Host, 
            options);
        
        return continuation(request, context);
    }
}
```

---

### 2.4 Presentation Layer (API Endpoints)

**Path:** `src/Services/[ServiceName]/Api/[ServiceName].Api/`

#### 2.4.1 Minimal API với Carter

```csharp
// src/Services/Catalog/Api/Catalog.Api/Endpoints/CreateProduct.cs
namespace Catalog.Api.Endpoints;

public sealed class CreateProduct : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost(ApiRoutes.Product.Create, HandleCreateProductAsync)
            .WithTags(ApiRoutes.Product.Tags)
            .WithName(nameof(CreateProduct))
            .WithMultipartForm<CreateProductRequest>()
            .Produces<ApiCreatedResponse<Guid>>(StatusCodes.Status201Created)
            .ProducesProblem(StatusCodes.Status400BadRequest)
            .DisableAntiforgery()
            .RequireAuthorization();
    }
    
    private async Task<ApiCreatedResponse<Guid>> HandleCreateProductAsync(
        ISender sender,
        IMapper mapper,
        IHttpContextAccessor httpContext,
        [FromForm] CreateProductRequest req)
    {
        if (req == null) throw new ClientValidationException(MessageCode.BadRequest);
        
        // Map Request to DTO
        var dto = mapper.Map<CreateProductDto>(req);
        
        // Handle file uploads
        if (req.ImageFiles != null && req.ImageFiles.Count > 0)
        {
            dto.UploadImages ??= new();
            foreach (var file in req.ImageFiles!)
            {
                using var ms = new MemoryStream();
                await file.CopyToAsync(ms);
                dto.UploadImages.Add(new UploadFileBytes
                {
                    FileName = file.FileName,
                    Bytes = ms.ToArray(),
                    ContentType = file.ContentType
                });
            }
        }
        
        // Get current user
        var currentUser = httpContext.GetCurrentUser();
        var command = new CreateProductCommand(dto, Actor.User(currentUser.Email));
        
        // Execute command via MediatR
        var result = await sender.Send(command);
        
        return new ApiCreatedResponse<Guid>(result);
    }
}
```

#### 2.4.2 Dependency Injection Setup

```csharp
// src/Services/Catalog/Api/Catalog.Api/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApiServices(
        this IServiceCollection services, IConfiguration cfg)
    {
        services.AddDistributedTracing(cfg);
        services.AddSerilogLogging(cfg);
        services.AddCarter(); // Minimal API framework
        
        // Health Checks
        var dbType = cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.DbType}"];
        var conn = cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.Database}"];
        
        switch (dbType)
        {
            case DatabaseType.SqlServer:
                services.AddHealthChecks().AddSqlServer(connectionString: conn!);
                break;
            case DatabaseType.MySql:
                services.AddHealthChecks().AddMySql(connectionString: conn!);
                break;
            case DatabaseType.PostgreSql:
                services.AddHealthChecks().AddNpgSql(connectionString: conn!);
                break;
        }
        
        services.AddHttpContextAccessor();
        services.AddAuthenticationAndAuthorization(cfg);
        services.AddSwaggerServices(cfg);
        services.AddAutoMapper(Assembly.GetExecutingAssembly());
        
        return services;
    }
    
    public static WebApplication UseApi(this WebApplication app)
    {
        app.UseSerilogReqLogging();
        app.UsePrometheusEndpoint();
        app.MapCarter(); // Map Carter endpoints
        app.UseExceptionHandler(options => { });
        app.UseHealthChecks("/health");
        
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseSwaggerApi();
        
        return app;
    }
}
```

#### 2.4.3 Program.cs Setup

```csharp
// src/Services/Catalog/Api/Catalog.Api/Program.cs
using Catalog.Api;
using Catalog.Application;
using Catalog.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// Add services - layered approach
builder.Services
    .AddApplicationServices()      // Application layer
    .AddInfrastructureServices(builder.Configuration)  // Infrastructure layer
    .AddApiServices(builder.Configuration);           // Presentation layer

var app = builder.Build();

// Configure pipeline
app.UseApi();           // Presentation (middleware)
app.UseInfrastructure(); // Infrastructure (if any)

app.Run();
```

---

### 2.5 Background Workers

#### 2.5.1 Outbox Pattern Processor

**Mục đích:** Đảm bảo eventual consistency với message bus.

```csharp
// src/Services/Catalog/Worker/Catalog.Worker.Outbox/BackgroundServices/OutboxBackgroundService.cs
public class OutboxBackgroundService(
    IServiceProvider serviceProvider,
    ILogger<OutboxBackgroundService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = serviceProvider.CreateScope();
                var processor = scope.ServiceProvider.GetRequiredService<OutboxProcessor>();
                
                await processor.ProcessOutboxMessagesAsync(stoppingToken);
                
                await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error in OutboxBackgroundService");
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }
    }
}

// src/Services/Catalog/Worker/Catalog.Worker.Outbox/Processors/OutboxProcessor.cs
public class OutboxProcessor(
    IOutboxRepository outboxRepository,
    IPublishEndpoint publishEndpoint,
    ILogger<OutboxProcessor> logger)
{
    public async Task ProcessOutboxMessagesAsync(CancellationToken cancellationToken)
    {
        // 1. Claim messages
        var messages = await outboxRepository.GetAndClaimMessagesAsync(
            batchSize: 20, cancellationToken);
        
        if (!messages.Any())
        {
            return;
        }
        
        // 2. Publish to message bus
        foreach (var message in messages)
        {
            try
            {
                var integrationEvent = JsonSerializer.Deserialize(
                    message.Payload!,
                    Type.GetType(message.EventType!));
                
                await publishEndpoint.Publish(integrationEvent, cancellationToken);
                
                // 3. Mark as processed
                message.ProcessedOnUtc = DateTimeOffset.UtcNow;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to process outbox message {MessageId}", message.Id);
                message.IncrementAttempt();
            }
        }
        
        // 4. Update messages
        await outboxRepository.UpdateMessagesAsync(messages, cancellationToken);
    }
}
```

#### 2.5.2 Message Bus Consumers

**Mục đích:** Handle integration events từ other services.

```csharp
// src/Services/Catalog/Worker/Catalog.Worker.Consumer/EventHandlers/Integrations/StockChangedEventHandler.cs
public class StockChangedEventHandler(
    IDocumentSession session,
    ILogger<StockChangedEventHandler> logger) 
    : IConsumer<StockChangedIntegrationEvent>
{
    public async Task Consume(ConsumeContext<StockChangedIntegrationEvent> context)
    {
        var @event = context.Message;
        
        try
        {
            // Update product status based on stock
            var product = await session.LoadAsync<ProductEntity>(@event.ProductId);
            if (product == null)
            {
                logger.LogWarning("Product {ProductId} not found", @event.ProductId);
                return;
            }
            
            var newStatus = @event.TotalStock > 0 
                ? ProductStatus.InStock 
                : ProductStatus.OutOfStock;
            
            if (product.Status != newStatus)
            {
                product.ChangeStatus(newStatus, "System");
                session.Store(product);
                await session.SaveChangesAsync();
                logger.LogInformation("Updated product {ProductId} status to {Status}", 
                                   @event.ProductId, newStatus);
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Error processing StockChangedIntegrationEvent");
            throw;
        }
    }
}
```

---

## 3. Đánh Giá Nguyên Tắc SOLID và Dependency Rule

### 3.1 Dependency Rule (Clean Architecture)

**Quy tắc:** Dependencies chỉ được chạy từ outside → inside

```
┌─────────────────────────────────────────────────────────┐
│              Presentation Layer                        │
│         (API, gRPC, Controllers)                    │
└───────────────────────┬─────────────────────────────┘
                        │ depends on
                        ▼
┌─────────────────────────────────────────────────────────┐
│              Application Layer                        │
│         (Use Cases, CQRS, DTOs)                    │
└───────────────────────┬─────────────────────────────┘
                        │ depends on
                        ▼
┌─────────────────────────────────────────────────────────┐
│                Domain Layer                           │
│         (Entities, Value Objects, Events)             │
└───────────────────────┬─────────────────────────────┘
                        │ depends on
                        ▼
┌─────────────────────────────────────────────────────────┐
│             Infrastructure Layer                       │
│    (DB, External Services, Repositories)              │
└─────────────────────────────────────────────────────────┘
```

**Đánh giá:**

| Layer | Depend on | Tuân thủ | Ghi chú |
|-------|-----------|-----------|----------|
| Presentation → Application | ✅ | Có | ISender, IMapper |
| Presentation → Infrastructure | ❌ | Không | Không có direct dependency |
| Application → Domain | ✅ | Có | Entities, Value Objects, Domain Events |
| Application → Infrastructure | ❌ | Không | Qua Interfaces |
| Infrastructure → Domain | ✅ | Có | Entities, Aggregates |
| Domain → Application/Infrastructure | ❌ | Không | Zero dependencies |

### 3.2 SOLID Principles

#### 3.2.1 Single Responsibility Principle (SRP)

**✅ TUÂN THỦ:**

```csharp
// Mỗi class có một trách nhiệm duy nhất

// Command Handler - chỉ xử lý business logic
public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, Guid> { }

// Validator - chỉ validate input
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand> { }

// Repository - chỉ thao tác data
public class OutboxRepository : IOutboxRepository { }

// Service - chỉ giao tiếp với external systems
public class MinIOCloudService : IMinIOCloudService { }
```

#### 3.2.2 Open/Closed Principle (OCP)

**✅ TUÂN THỦ:**

```csharp
// Mở rộng bằng cách thêm mới thay vì sửa code hiện có

// Thêm mới command/query tự động được register
services.AddMediatR(config =>
{
    config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
});

// Scan và register repositories tự động
services.Scan(s => s
    .FromAssemblyOf<InfrastructureMarker>()
    .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
    .AsImplementedInterfaces()
    .WithScopedLifetime());

// Thêm mới behaviors không cần sửa code hiện có
config.AddOpenBehavior(typeof(ValidationBehavior<,>));
config.AddOpenBehavior(typeof(LoggingBehavior<,>));
```

#### 3.2.3 Liskov Substitution Principle (LSP)

**✅ TUÂN THỦ:**

```csharp
// Subclasses có thể thay thế parent classes

public abstract class Aggregate<TId> : Entity<TId>, IAggregate<TId>
{
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    public void AddDomainEvent(IDomainEvent domainEvent) { }
}

// Tất cả aggregates đều có thể được xử lý theo cách giống nhau
public void DispatchDomainEvents(IEnumerable<AggregateRoot> aggregates)
{
    foreach (var aggregate in aggregates)
    {
        foreach (var domainEvent in aggregate.DomainEvents)
        {
            _publisher.Publish(domainEvent);
        }
        aggregate.ClearDomainEvents();
    }
}
```

#### 3.2.4 Interface Segregation Principle (ISP)

**✅ TUÂN THỦ:**

```csharp
// Interfaces nhỏ, focused

public interface ICommandHandler<in TCommand, TResponse> { }

public interface IQueryHandler<in TQuery, TResponse> { }

public interface INotificationHandler<in TNotification> { }

// Repository interfaces nhỏ, focused
public interface IOutboxRepository
{
    Task<bool> AddMessageAsync(OutboxMessageEntity message, ...);
    Task<List<OutboxMessageEntity>> GetAndClaimMessagesAsync(...);
}

public interface IProductRepository
{
    Task<ProductEntity?> GetByIdAsync(Guid id, ...);
    Task<List<ProductEntity>> GetByCategoryAsync(Guid categoryId, ...);
}
```

#### 3.2.5 Dependency Inversion Principle (DIP)

**✅ TUÂN THỦ:**

```csharp
// High-level modules (Application) không phụ thuộc low-level modules (Infrastructure)
// Cả hai đều phụ thuộc vào abstractions (Interfaces)

// Application layer - Interface ở đây
namespace Catalog.Application.Repositories;
public interface IOutboxRepository { ... }

// Infrastructure layer - Implementation ở đây
namespace Catalog.Infrastructure.Repositories;
public class OutboxRepository : IOutboxRepository { ... }

// Dependency Injection trong Presentation layer
builder.Services
    .AddApplicationServices()      // Register interfaces
    .AddInfrastructureServices(...) // Register implementations
```

---

## 4. Sơ Đồ Luồng Dữ Liệu và Dependency Flow

### 4.1 Request Flow (HTTP API)

```
Client (React App)
    │
    │ HTTP POST /api/products with multipart form data
    ▼
┌─────────────────────────────────────────────────────────┐
│ Presentation Layer (Catalog.Api)                        │
│ ┌──────────────────────────────────────────────────────┐│
│ │ CreateProduct Endpoint (Carter Module)              ││
│ │ - Validates request size                             ││
│ │ - Handles file uploads                              ││
│ │ - Maps Request → DTO                               ││
│ │ - Gets User Context                                ││
│ │ - Creates Command                                  ││
│ └────────────────────┬───────────────────────────────┘│
└───────────────────────┼───────────────────────────────┘
                        │ ISender.Send(command)
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Application Layer (Catalog.Application)                  │
│ ┌──────────────────────────────────────────────────────┐│
│ │ MediatR Pipeline                                  ││
│ │ ┌─────────────┐                                 ││
│ │ │ Validation  │ → Validate with FluentValidation      ││
│ │ │  Behavior   │                                 ││
│ │ └──────┬──────┘                                 ││
│ │        ↓                                        ││
│ │ ┌─────────────┐                                 ││
│ │ │  Logging   │ → Log request details               ││
│ │ │  Behavior   │                                 ││
│ │ └──────┬──────┘                                 ││
│ │        ↓                                        ││
│ │ ┌──────────────────────┐                          ││
│ │ │ CommandHandler      │                          ││
│ │ │ - Validate Domain    │                          ││
│ │ │ - Create Entity     │                          ││
│ │ │ - Upload Images     │                          ││
│ │ │ - Persist Data      │                          ││
│ │ └────────┬────────────┘                          ││
│ └──────────┼──────────────────────────────────────────┘│
└───────────┼────────────────────────────────────────────┘
            │ IDocumentSession (Infrastructure)
            ▼
┌─────────────────────────────────────────────────────────┐
│ Infrastructure Layer (Catalog.Infrastructure)             │
│ ┌──────────────────────────────────────────────────────┐│
│ │ Marten (PostgreSQL Document Store)                  ││
│ │ - Serialize entity to JSON                          ││
│ │ - Store in PostgreSQL                              ││
│ │ - Return on success                               ││
│ └────────────────────┬───────────────────────────────┘│
│                      │                             │
│        ┌─────────────┴────────────┐              │
│        ▼                          ▼              │
│ ┌─────────────────┐      ┌────────────────┐       │
│ │   MinIO         │      │   EF Core     │       │
│ │ (Object Storage) │      │ (Relational)  │       │
│ └─────────────────┘      └────────────────┘       │
└─────────────────────────────────────────────────────────┘
            │
            │ IDomainEvent.Dispatch()
            ▼
┌─────────────────────────────────────────────────────────┐
│ Application Layer (Event Handlers)                     │
│ ┌──────────────────────────────────────────────────────┐│
│ │ UpsertedProductDomainEventHandler                   ││
│ │ - Creates IntegrationEvent                        ││
│ │ - Saves to Outbox                               ││
│ └────────────────────┬───────────────────────────────┘│
└──────────────────────┼───────────────────────────────┘
                       │ IOutboxRepository
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Infrastructure Layer                                │
│ ┌──────────────────────────────────────────────────────┐│
│ │ OutboxRepository → Saves to Marten               ││
│ └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
                        │
                        │ (background process)
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Worker (Catalog.Worker.Outbox)                       │
│ ┌──────────────────────────────────────────────────────┐│
│ │ OutboxProcessor                                   ││
│ │ - Reads unprocessed messages                       ││
│ │ - Publishes to RabbitMQ                          ││
│ │ - Marks as processed                              ││
│ └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
                        │
                        │ MassTransit
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Other Services (Search, Inventory, etc.)             │
│ ┌──────────────────────────────────────────────────────┐│
│ │ IntegrationEventHandlers                           ││
│ │ - Consumes from RabbitMQ                         ││
│ │ - Updates local state                             ││
│ └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### 4.2 Dependency Flow Diagram

```
                    ┌─────────────────────────────────────────────┐
                    │          Presentation Layer                 │
                    │  ┌───────────────────────────────────┐  │
                    │  │   Minimal API (Carter)          │  │
                    │  │   - Endpoints                     │  │
                    │  │   - Authorization                 │  │
                    │  │   - Validation                   │  │
                    │  └────────────────┬──────────────────┘  │
                    └───────────────────┼──────────────────────┘
                                        │ depends on
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Application Layer                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              MediatR (CQRS)                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐│  │
│  │  │  Commands     │  │   Queries    │  │ Events   ││  │
│  │  │  - Create    │  │   - Get      │  │ Handlers ││  │
│  │  │  - Update    │  │   - Search   │  │         ││  │
│  │  │  - Delete    │  │             │  │         ││  │
│  │  └──────────────┘  └──────────────┘  └──────────┘│  │
│  │         │                   │                   │         │  │
│  │         ▼                   ▼                   ▼         │  │
│  │  ┌───────────────────────────────────────────────┐   │  │
│  │  │         ValidationBehavior                 │   │  │
│  │  │         LoggingBehavior                   │   │  │
│  │  └───────────────────────────────────────────┘   │  │
│  └────────────────────────┬──────────────────────────────┘  │
└───────────────────────────┼──────────────────────────────────┘
                            │ depends on
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Domain Layer                               │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Aggregates (Root Entities)                   │  │
│  │  - ProductEntity                                        │  │
│  │  - OrderEntity                                          │  │
│  │  - ShoppingCartEntity                                    │  │
│  │         │                                                 │  │
│  │         ▼                                                 │  │
│  │  ┌───────────────────────────────────────────────┐         │  │
│  │  │    Domain Events                            │         │  │
│  │  │  - UpsertedProductDomainEvent               │         │  │
│  │  │  - OrderCreatedDomainEvent                  │         │  │
│  │  │  - StockChangedDomainEvent                  │         │  │
│  │  └───────────────────────────────────────────────┘         │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────────┘
                            │ depends on
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                             │
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐│
│  │  Repositories  │  │   Database    │  │  External      ││
│  │  - OutboxRepo │  │   - Marten    │  │  Services      ││
│  │  - ProductRepo│  │   - EF Core   │  │  - MinIO       ││
│  │               │  │               │  │  - gRPC Client ││
│  └───────┬───────┘  └───────┬────────┘  └────────┬────────┘│
│          │                   │                     │             │
│          │                   │                     │             │
│          └───────────────────┴─────────────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Phân Tích Dependency Injection và Inversion of Control

### 5.1 DI Container Structure

**Đa số services sử dụng .NET DI Container với pattern:**

```csharp
// Program.cs - Layered DI Registration
builder.Services
    .AddApplicationServices()           // Register CQRS, Validators, Mappings
    .AddInfrastructureServices(...)     // Register DB, Repositories, External Services
    .AddApiServices(...);            // Register API, Middleware, Auth
```

### 5.2 Service Lifetimes

```csharp
// Scoped - Request-based (most common)
services.Scan(s => s
    .FromAssemblyOf<InfrastructureMarker>()
    .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
    .AsImplementedInterfaces()
    .WithScopedLifetime());

// Singleton - App-wide (granted services like Configuration)
services.AddSingleton<GrpcClientCfg>();
services.AddSingleton<IMinIOCloudService, MinIOCloudService>();

// Transient - New instance each time (rarely used)
services.AddTransient<IValidator<CreateProductCommand>, CreateProductCommandValidator>();
```

### 5.3 Auto-Registration Patterns

#### 5.3.1 Repository Scan (Scrutor)

```csharp
// Auto-register all repositories
services.Scan(s => s
    .FromAssemblyOf<InfrastructureMarker>()
    .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
    .UsingRegistrationStrategy(Scrutor.RegistrationStrategy.Skip)
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

**Ưu điểm:**
- Thêm mới repository không cần sửa DI code
- Giảm boilerplate code
- Prevent duplicate registration

#### 5.3.2 MediatR Assembly Registration

```csharp
// Auto-register all handlers, behaviors, validators
services.AddMediatR(config =>
{
    config.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    config.AddOpenBehavior(typeof(ValidationBehavior<,>));
    config.AddOpenBehavior(typeof(LoggingBehavior<,>));
});
```

**Ưu điểm:**
- Tự động tìm và register tất cả handlers
- Pipeline behaviors được apply cho tất cả commands/queries

#### 5.3.3 AutoMapper Profile Registration

```csharp
// Auto-register all profiles from assembly
services.AddAutoMapper(Assembly.GetExecutingAssembly());
```

### 5.4 Factory Pattern for Database Providers

```csharp
// src/Services/Order/Worker/Order.Woker.Outbox/Factories/DatabaseProviderFactory.cs
public static class DatabaseProviderFactory
{
    public static IDatabaseProvider Create(IConfiguration cfg)
    {
        var dbType = cfg[$"{ConnectionStringsCfg.Section}:{ConnectionStringsCfg.DbType}"];
        
        return dbType switch
        {
            DatabaseType.SqlServer => new SqlServerDatabaseProvider(cfg),
            DatabaseType.MySql => new MySqlDatabaseProvider(cfg),
            DatabaseType.PostgreSql => new PostgreSqlDatabaseProvider(cfg),
            _ => throw new NotSupportedException($"Database type {dbType} is not supported")
        };
    }
}

// Usage
var provider = DatabaseProviderFactory.Create(configuration);
var processor = new OutboxProcessor(provider);
```

### 5.5 Interceptor Pattern (gRPC)

```csharp
// API Key authentication interceptor
public class GrpcApiKeyInterceptor : Interceptor
{
    private readonly GrpcClientCfg _cfg;
    
    public GrpcApiKeyInterceptor(GrpcClientCfg cfg)
    {
        _cfg = cfg;
    }
    
    public override AsyncUnaryCall<TRequest, TResponse> UnaryAsyncUnary<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        // Add API key to all gRPC calls
        var metadata = new Metadata
        {
            { "X-API-Key", _cfg.ApiKey }
        };
        
        var options = context.Options.WithHeaders(metadata);
        var newContext = new ClientInterceptorContext<TRequest, TResponse>(
            context.Method, 
            context.Host, 
            options);
        
        return continuation(request, newContext);
    }
}

// Register with client
services.AddGrpcClient<CatalogGrpc.CatalogGrpcClient>(options =>
{
    options.Address = new Uri(grpcCfg.CatalogServiceUrl!);
}).AddInterceptor<GrpcApiKeyInterceptor>();
```

---

## 6. Đánh Giá Ưu Điểm, Nhược Điểm và Các Vi Phạm

### 6.1 Ưu Điểm

#### 6.1.1 Clean Architecture Compliance ✅

**Tách biệt rõ ràng các layer:**
- Domain layer độc lập hoàn toàn (zero dependencies)
- Application layer chứa business logic orchestration
- Infrastructure layer chỉ chứa technical implementations
- Presentation layer chỉ chứa HTTP/gRPC handling

**Code ví dụ:**

```csharp
// Domain layer - zero dependencies
public sealed record UpsertedProductDomainEvent(...) : IDomainEvent;

// Application layer - depends on Domain
public class UpsertedProductDomainEventHandler : INotificationHandler<UpsertedProductDomainEvent>;

// Infrastructure layer - implements Application interfaces
public class OutboxRepository : IOutboxRepository;

// Presentation layer - uses Application layer via ISender
public async Task<ApiCreatedResponse<Guid>> HandleCreateProductAsync(
    ISender sender, CreateProductRequest req);
```

#### 6.1.2 SOLID Principles ✅

**Đã tuân thủ tốt:**

- **SRP:** Mỗi class có một trách nhiệm rõ ràng
- **OCP:** Mở rộng bằng cách thêm mới (MediatR, Scrutor)
- **LSP:** Aggregates và Entities có thể thay thế nhau
- **ISP:** Interfaces nhỏ, focused (ICommand, IQuery, INotification)
- **DIP:** Dependency Injection với interfaces

#### 6.1.3 Event-Driven Architecture ✅

**Domain Events:**
```csharp
// 1. Entity emits domain event
public void Publish(string performedBy)
{
    Published = true;
    LastModifiedBy = performedBy;
    AddDomainEvent(new ProductPublishedDomainEvent(Id, Name, Sku));
}

// 2. Handler creates integration event and saves to outbox
public async Task Handle(ProductPublishedDomainEvent evt, ...)
{
    var integrationEvent = new ProductPublishedIntegrationEvent(...);
    await outboxRepository.AddMessageAsync(integrationEvent);
}

// 3. Background worker publishes to message bus
await publishEndpoint.Publish(integrationEvent);
```

**Ưu điểm:**
- Loosely coupled
- Eventual consistency
- Reliable messaging (outbox pattern)

#### 6.1.4 CQRS Pattern ✅

**Separation of Read/Write:**

```csharp
// Write - Commands
public record CreateProductCommand(...) : ICommand<Guid>;
public record UpdateProductCommand(...) : ICommand<Unit>;
public record DeleteProductCommand(...) : ICommand<Unit>;

// Read - Queries
public record GetProductsQuery(...) : IQuery<GetProductsResult>;
public record GetProductByIdQuery(...) : IQuery<ProductDto>;

// Independent handlers
public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, Guid> { }
public class GetProductsQueryHandler : IQueryHandler<GetProductsQuery, GetProductsResult> { }
```

**Ưu điểm:**
- Clear separation of concerns
- Optimized for different use cases
- Independent scaling possibility

#### 6.1.5 Outbox Pattern ✅

**Reliable message delivery:**

```csharp
// 1. Save domain event + outbox message in same transaction
await session.BeginTransactionAsync();
session.Store(product);
session.Store(outboxMessage);
await session.SaveChangesAsync();

// 2. Background worker processes outbox
var messages = await outboxRepository.GetAndClaimMessagesAsync(batchSize);
foreach (var message in messages)
{
    await publishEndpoint.Publish(message.Payload);
    message.ProcessedOnUtc = DateTimeOffset.UtcNow;
}
await outboxRepository.UpdateMessagesAsync(messages);
```

**Ưu điểm:**
- Atomic transaction (domain state + outbox)
- Retry mechanism with exponential backoff
- No lost messages

#### 6.1.6 Multiple Database Technologies ✅

**Polyglot persistence:**

```csharp
// Catalog - Marten (PostgreSQL document store)
services.AddMarten(opts =>
{
    opts.Connection(connectionString);
});

// Order/Inventory - EF Core (Relational)
services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseNpgsql(connectionString);
});

// Basket/Discount - MongoDB
services.AddSingleton<IMongoClient>(sp =>
{
    var settings = MongoClientSettings.FromConnectionString(connectionString);
    return new MongoClient(settings);
});
```

**Ưu điểm:**
- Choose right DB for each use case
- Document store for flexible schemas (Catalog)
- Relational for complex relationships (Order)
- MongoDB for high write volume (Basket)

### 6.2 Nhược Điểm

#### 6.2.1 High Code Volume 📊

**Issue:** Clean Architecture tạo nhiều boilerplate code

**Ví dụ - Tạo 1 entity cần:**

1. **Domain:**
   - Entity class
   - Aggregate base
   - Domain events
   - Value objects

2. **Application:**
   - DTOs (Create, Update, Response)
   - Command (Create, Update, Delete)
   - Query (Get, GetAll, Search)
   - Validators
   - Mapping profiles

3. **Infrastructure:**
   - Repository interface
   - Repository implementation
   - EF/Marten configuration

4. **Presentation:**
   - Request model
   - Endpoint class
   - Response wrapper

**Tổng:** ~10-15 files cho 1 entity

**Giải pháp đề xuất:**
- Sử dụng source generators
- Code templates
- Scaffolding tools

#### 6.2.2 Over-Engineering for Simple CRUD 🤔

**Issue:** Đối với simple CRUD, Clean Architecture có thể overkill

**Ví dụ:**

```csharp
// Phức tạp cho simple get
public record GetProductByIdQuery(Guid Id) : IQuery<ProductDto>;

public class GetProductByIdQueryHandler : IQueryHandler<GetProductByIdQuery, ProductDto>
{
    public async Task<ProductDto> Handle(GetProductByIdQuery query, ...)
    {
        var product = await session.LoadAsync<ProductEntity>(query.Id);
        return mapper.Map<ProductDto>(product);
    }
}

// Endpoint
public async Task<ProductDto> HandleGetProductById(Guid id)
{
    return await sender.Send(new GetProductByIdQuery(id));
}
```

**Giải pháp đề xuất:**
- Generic CRUD endpoints
- Repository pattern cho simple cases
- Minimal Architecture cho simple services

#### 6.2.3 Complexity in Event Handling 🔄

**Issue:** Tracking message flow khó khăn

**Đường đi của 1 message:**

```
Domain Entity 
  → Domain Event 
  → Domain Event Handler 
  → Integration Event 
  → Outbox Message 
  → Outbox Processor 
  → Message Bus (RabbitMQ) 
  → Integration Event Handler (Other Service)
  → Inbox Message (Optional)
  → Application Command/Query
  → Repository
  → Database
```

**Giải pháp đề xuất:**
- Distributed tracing (Jaeger/Tempo)
- Correlation IDs across services
- Event sourcing audit logs

#### 6.2.4 Performance Considerations ⚡

**Issue:** Multiple layers tạo overhead

**Ví dụ:**

```csharp
// 1. Controller/Endpoint (Presentation)
// 2. Mapper (DTO → Command)
// 3. Validation Behavior
// 4. Logging Behavior
// 5. Command Handler (Application)
// 6. Mapper (Command → Entity)
// 7. Repository (Infrastructure)
// 8. Database

// Tối thiểu 8 layers cho 1 request
```

**Giải pháp đề xuất:**
- Benchmark để đo overhead thực tế
- Caching để giảm DB calls
- Async/await đúng cách
- Avoid over-mapping

### 6.3 Các Vi Phạm (Violations)

#### 6.3.1 Infrastructure Leaking into Application ⚠️

**Issue:** Một số application services có dependencies vào Marten/EF specific types

```csharp
// VÍ DỤ KHÔNG TỐT
public class CreateProductCommandHandler(
    IDocumentSession session,  // Marten-specific!
    ...)
{
    // Application layer phụ thuộc vào Marten type
}
```

**Đúng hơn:**

```csharp
// Đúng - Application interface
namespace Catalog.Application.Repositories;
public interface IProductRepository
{
    Task<ProductEntity?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<ProductEntity>> GetByCategoryAsync(Guid categoryId, CancellationToken ct = default);
}

// Infrastructure implementation
public class ProductRepository : IProductRepository
{
    private readonly IDocumentSession _session;
    
    public async Task<ProductEntity?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        return await _session.LoadAsync<ProductEntity>(id, token: ct);
    }
}
```

**Thực tế:** Dự án có một số handlers phụ thuộc trực tiếp vào `IDocumentSession` thay vì repository interface.

#### 6.3.2 Domain Services Missing in Domain 📝

**Issue:** Domain Services nên nằm trong Domain layer nhưng thực tế lại ở Application layer

```csharp
// Nên ở Domain Layer
namespace Catalog.Domain.Services;
public interface IPriceCalculator
{
    decimal CalculateDiscountedPrice(decimal originalPrice, decimal discountPercentage);
}

// Thực tế ở Application Layer
namespace Catalog.Application.Services;
public interface ICatalogGrpcService
{
    Task<GetAllProductsResponse> GetAllProductsAsync(...);
}
```

**Lý do:** External services (gRPC, HTTP) phải ở Infrastructure layer, Interface ở Application layer. Nhưng Domain Services (pure business logic) nên ở Domain layer.

#### 6.3.3 Anemic Domain Models 🫀

**Issue:** Một số entities thiếu domain behaviors

```csharp
// VÍ DỤ KHÔNG TỐT - Anemic model
public class ProductEntity
{
    public decimal Price { get; set; }
    public decimal? SalePrice { get; set; }
    public decimal Discount => Price - (SalePrice ?? Price);
}

// Tốt hơn - Encapsulated behavior
public class ProductEntity
{
    public decimal Price { get; private set; }
    public decimal? SalePrice { get; private set; }
    
    public decimal GetDiscount()
    {
        return Price - (SalePrice ?? Price);
    }
    
    public void ApplyDiscount(decimal salePrice, string performedBy)
    {
        if (salePrice < 0)
            throw new DomainException("Sale price cannot be negative");
        if (salePrice >= Price)
            throw new DomainException("Sale price must be less than original price");
        
        SalePrice = salePrice;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
    }
}
```

**Thực tế:** Dự án có làm tốt một phần nhưng vẫn có các properties có public setter mà nên private.

#### 6.3.4 Direct Database Access in Command Handlers 🗄️

**Issue:** Command handlers phụ thuộc trực tiếp vào `IDocumentSession`/`DbContext`

```csharp
// Hiện tại trong dự án
public class CreateProductCommandHandler(
    IDocumentSession session,  // Direct DB access
    ...)
{
    public async Task<Guid> Handle(CreateProductCommand command, ...)
    {
        await session.BeginTransactionAsync(cancellationToken);
        session.Store(entity);
        await session.SaveChangesAsync(cancellationToken);
    }
}

// Nên dùng Repository pattern
public class CreateProductCommandHandler(
    IProductRepository repository,  // Abstracted
    ...)
{
    public async Task<Guid> Handle(CreateProductCommand command, ...)
    {
        await repository.AddAsync(entity, cancellationToken);
    }
}
```

---

## 7. Đề Xuất Cải Tiến và Best Practices

### 7.1 Giảm Boilerplate Code

#### 7.1.1 Source Generators

```csharp
// Tạo file Product.g.cs với Source Generator
[GenerateDto("Product")]
public partial class ProductDto;

// Generator sẽ tự tạo:
// - CreateProductDto
// - UpdateProductDto
// - ProductDto
// - Mapping Profile
// - Validators
```

#### 7.1.2 Generic CRUD Endpoints

```csharp
// Tạo generic endpoint cho CRUD operations
public static class GenericEndpoints<TEntity, TDto, TCreateDto, TUpdateDto>
    where TEntity : Aggregate<Guid>
    where TDto : class
    where TCreateDto : class
    where TUpdateDto : class
{
    public static void MapGenericEndpoints(IEndpointRouteBuilder app, string basePath)
    {
        app.MapGet($"{basePath}/{{id}}", GetByIdAsync)
            .WithName($"Get{typeof(TEntity).Name}ById");
        
        app.MapGet(basePath, GetAllAsync)
            .WithName($"GetAll{typeof(TEntity).Name}s");
        
        app.MapPost(basePath, CreateAsync)
            .WithName($"Create{typeof(TEntity).Name}");
        
        app.MapPut($"{basePath}/{{id}}", UpdateAsync)
            .WithName($"Update{typeof(TEntity).Name}");
        
        app.MapDelete($"{basePath}/{{id}}", DeleteAsync)
            .WithName($"Delete{typeof(TEntity).Name}");
    }
    
    // Implementations...
}

// Usage
GenericEndpoints<ProductEntity, ProductDto, CreateProductDto, UpdateProductDto>
    .MapGenericEndpoints(app, "/api/products");
```

### 7.2 Domain Modeling Improvements

#### 7.2.1 Rich Domain Models

```csharp
// Thêm domain behaviors thay vì chỉ properties

public class OrderEntity : Aggregate<Guid>
{
    // State
    public OrderStatus Status { get; private set; }
    public List<OrderItemEntity> Items { get; private set; }
    
    // Behaviors
    public void Confirm(string performedBy)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Order must be pending to confirm");
        
        Status = OrderStatus.Confirmed;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new OrderConfirmedDomainEvent(Id));
    }
    
    public void Ship(string trackingNumber, string performedBy)
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException("Order must be confirmed to ship");
        
        if (string.IsNullOrWhiteSpace(trackingNumber))
            throw new DomainException("Tracking number is required");
        
        Status = OrderStatus.Shipped;
        TrackingNumber = trackingNumber;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new OrderShippedDomainEvent(Id, trackingNumber));
    }
    
    public void Cancel(string reason, string performedBy)
    {
        if (Status == OrderStatus.Delivered)
            throw new DomainException("Delivered order cannot be cancelled");
        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Order already cancelled");
        
        Status = OrderStatus.Cancelled;
        CancellationReason = reason;
        LastModifiedBy = performedBy;
        LastModifiedOnUtc = DateTimeOffset.UtcNow;
        
        AddDomainEvent(new OrderCancelledDomainEvent(Id, reason));
    }
}
```

#### 7.2.2 Domain Services in Domain Layer

```csharp
// src/Services/Order/Core/Order.Domain/Services/IPricingCalculator.cs
namespace Order.Domain.Services;

public interface IPricingCalculator
{
    decimal CalculateOrderTotal(List<OrderItem> items);
    decimal CalculateTax(decimal subtotal, string countryCode);
    decimal ApplyDiscount(decimal total, DiscountCode discount);
}

// Implementation
public class PricingCalculator : IPricingCalculator
{
    private readonly ITaxRateProvider _taxRateProvider;
    
    public PricingCalculator(ITaxRateProvider taxRateProvider)
    {
        _taxRateProvider = taxRateProvider;
    }
    
    public decimal CalculateOrderTotal(List<OrderItem> items)
    {
        var subtotal = items.Sum(i => i.Price * i.Quantity);
        var tax = CalculateTax(subtotal, "US");
        return subtotal + tax;
    }
    
    public decimal CalculateTax(decimal subtotal, string countryCode)
    {
        var rate = _taxRateProvider.GetTaxRate(countryCode);
        return subtotal * rate / 100;
    }
    
    public decimal ApplyDiscount(decimal total, DiscountCode discount)
    {
        return total * (100 - discount.Percentage) / 100;
    }
}

// Use in Domain Entity
public class OrderEntity : Aggregate<Guid>
{
    private readonly IPricingCalculator _pricingCalculator;
    
    public decimal CalculateTotal()
    {
        return _pricingCalculator.CalculateOrderTotal(Items);
    }
}
```

### 7.3 Testing Improvements

#### 7.3.1 Unit Testing Domain Layer

```csharp
// Test domain logic trực tiếp, không cần mock
public class OrderEntityTests
{
    [Fact]
    public void Confirm_ShouldChangeStatusToConfirmed_WhenPending()
    {
        // Arrange
        var order = OrderEntity.Create(...);
        
        // Act
        order.Confirm("test-user");
        
        // Assert
        order.Status.Should().Be(OrderStatus.Confirmed);
        order.LastModifiedBy.Should().Be("test-user");
        order.DomainEvents.Should().ContainSingle(e => 
            e is OrderConfirmedDomainEvent);
    }
    
    [Fact]
    public void Confirm_ShouldThrow_WhenAlreadyShipped()
    {
        // Arrange
        var order = OrderEntity.Create(...);
        order.Ship("TRK123", "test-user");
        
        // Act & Assert
        Assert.Throws<DomainException>(() => 
            order.Confirm("test-user"));
    }
}
```

#### 7.3.2 Integration Testing with TestContainers

```csharp
// Test với real database
public class OrderRepositoryTests : IClassFixture<PostgresContainer>
{
    private readonly PostgresContainer _postgres;
    
    public OrderRepositoryTests(PostgresContainer postgres)
    {
        _postgres = postgres;
    }
    
    [Fact]
    public async Task AddAsync_ShouldPersistOrder()
    {
        // Arrange
        var repository = new OrderRepository(_postgres.CreateDbContext());
        var order = OrderEntity.Create(...);
        
        // Act
        await repository.AddAsync(order);
        
        // Assert
        var retrieved = await repository.GetByIdAsync(order.Id);
        retrieved.Should().NotBeNull();
        retrieved.Status.Should().Be(order.Status);
    }
}
```

### 7.4 Performance Optimizations

#### 7.4.1 Response Caching

```csharp
// Cache GET requests cho read-heavy endpoints
public class GetProducts : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/products", HandleGetProductsAsync)
            .CacheOutput("products-cache", builder =>
            {
                builder.Expire(TimeSpan.FromMinutes(5));
                builder.VaryByQuery("category", "page", "pageSize");
            });
    }
}
```

#### 7.4.2 Read Models (CQRS)

```csharp
// Tạo read-optimized models riêng biệt
public class ProductReadModel
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Thumbnail { get; set; }
    public decimal CurrentPrice { get; set; }
    public int TotalStock { get; set; }
    public List<string> Categories { get; set; }
    // Denormalized for fast queries
}

// Store trong Elasticsearch hoặc read-optimized DB
public interface IProductReadModelRepository
{
    Task<ProductReadModel?> GetByIdAsync(Guid id);
    Task<List<ProductReadModel>> SearchAsync(SearchFilter filter);
}
```

#### 7.4.3 Pagination

```csharp
// Efficient pagination implementation
public async Task<PaginatedResult<ProductDto>> GetProductsAsync(
    GetProductsQuery query)
{
    var pageNumber = query.Filter.PageNumber;
    var pageSize = query.Filter.PageSize;
    
    // Count query (can be cached)
    var totalCount = await session.Query<ProductEntity>()
        .CountAsync(cancellationToken);
    
    // Paged query
    var products = await session.Query<ProductEntity>()
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync(cancellationToken);
    
    return new PaginatedResult<ProductDto>
    {
        Items = mapper.Map<List<ProductDto>>(products),
        TotalCount = totalCount,
        PageNumber = pageNumber,
        PageSize = pageSize,
        TotalPages = (int)Math.Ceiling((double)totalCount / pageSize)
    };
}
```

### 7.5 Security Improvements

#### 7.5.1 Input Validation

```csharp
// FluentValidation với custom validators
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Dto.Name)
            .NotEmpty()
            .MaximumLength(200)
            .Must(BeValidProductName)
                .WithMessage("Product name contains invalid characters");
        
        RuleFor(x => x.Dto.Price)
            .GreaterThan(0)
            .LessThan(1000000);
        
        RuleFor(x => x.Dto.CategoryIds)
            .NotNull()
            .Must(ids => ids != null && ids.Count <= 10)
                .WithMessage("Max 10 categories allowed");
    }
    
    private bool BeValidProductName(string name)
    {
        // Custom validation logic
        return !Regex.IsMatch(name, @"[<>""'&]");
    }
}
```

#### 7.5.2 Output Sanitization

```csharp
// Sanitize outputs để prevent XSS
public class SanitizeBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        var response = await next(request, cancellationToken);
        
        if (response is string str)
        {
            return (TResponse)(object)HtmlEncoder.Default.Encode(str);
        }
        
        return response;
    }
}
```

### 7.6 Monitoring and Observability

#### 7.6.1 Structured Logging

```csharp
// Log với structured data
public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, Guid>
{
    private readonly ILogger<CreateProductCommandHandler> _logger;
    
    public async Task<Guid> Handle(CreateProductCommand command, ...)
    {
        _logger.LogInformation("Creating product with data: {@Command}", 
            new { Command = command.Dto });
        
        // ... logic ...
        
        _logger.LogInformation("Product created successfully: {ProductId}, {@Product}", 
            result.Id, 
            new { Product = entity });
        
        return result;
    }
}
```

#### 7.6.2 Distributed Tracing

```csharp
// Sử dụng OpenTelemetry cho distributed tracing
public class TracingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        using var activity = ActivitySource.StartActivity($"Handle_{typeof(TRequest).Name}");
        
        activity?.SetTag("request.type", typeof(TRequest).Name);
        activity?.SetTag("correlation.id", CorrelationContext.Current);
        
        try
        {
            var response = await next(request, cancellationToken);
            activity?.SetStatus(ActivityStatusCode.Ok);
            return response;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.SetTag("error.type", ex.GetType().Name);
            activity?.SetTag("error.message", ex.Message);
            throw;
        }
    }
}
```

### 7.7 Error Handling

#### 7.7.1 Global Exception Handler

```csharp
// src/Shared/BuildingBlocks/Exceptions/Handler/CustomExceptionHandler.cs
public class CustomExceptionHandler : IExceptionHandler
{
    public async ValueTask<ProblemDetails> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (statusCode, title, detail) = exception switch
        {
            ClientValidationException ex => (StatusCodes.Status400BadRequest, "Validation Error", ex.Message),
            NotFoundException ex => (StatusCodes.Status404NotFound, "Not Found", ex.Message),
            UnauthorizedException ex => (StatusCodes.Status401Unauthorized, "Unauthorized", ex.Message),
            NoPermissionException ex => (StatusCodes.Status403Forbidden, "Forbidden", ex.Message),
            DomainException ex => (StatusCodes.Status400BadRequest, "Domain Error", ex.Message),
            _ => (StatusCodes.Status500InternalServerError, "Internal Server Error", "An unexpected error occurred")
        };
        
        httpContext.Response.StatusCode = statusCode;
        
        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = detail,
            Type = exception.GetType().Name,
            Instance = httpContext.Request.Path
        };
        
        problemDetails.Extensions["correlationId"] = httpContext.TraceIdentifier;
        problemDetails.Extensions["timestamp"] = DateTimeOffset.UtcNow;
        
        return ValueTask.FromResult(problemDetails);
    }
}
```

### 7.8 Documentation

#### 7.8.1 OpenAPI/Swagger Configuration

```csharp
// Enhanced Swagger with examples and descriptions
public class CreateProductEndpointConfig
{
    public static OpenApiOperation GetOperation()
    {
        return new OpenApiOperation
        {
            Description = "Creates a new product with images",
            Tags = new List<OpenApiTag> { new() { Name = "Products" } },
            RequestBody = new OpenApiRequestBody
            {
                Description = "Product data with image uploads",
                Required = true,
                Content = new Dictionary<string, OpenApiMediaType>
                {
                    ["multipart/form-data"] = new OpenApiMediaType
                    {
                        Schema = new OpenApiSchema
                        {
                            Type = "object",
                            Properties = new Dictionary<string, OpenApiSchema>
                            {
                                ["name"] = new OpenApiSchema
                                {
                                    Type = "string",
                                    Example = "Premium Headphones",
                                    Description = "Product name",
                                    MinLength = 1,
                                    MaxLength = 200
                                },
                                ["price"] = new OpenApiSchema
                                {
                                    Type = "number",
                                    Format = "decimal",
                                    Example = 99.99,
                                    Minimum = 0.01
                                }
                            },
                            Required = new HashSet<string> { "name", "price" }
                        }
                    }
                }
            },
            Responses = new OpenApiResponses
            {
                ["201"] = new OpenApiResponse
                {
                    Description = "Product created successfully",
                    Content = new Dictionary<string, OpenApiMediaType>
                    {
                        ["application/json"] = new OpenApiMediaType
                        {
                            Schema = new OpenApiSchema
                            {
                                Reference = new OpenApiReference
                                {
                                    Type = ReferenceType.Schema,
                                    Id = "ApiCreatedResponse"
                                }
                            }
                        }
                    }
                }
            }
        };
    }
}
```

---

## Summary

### Kiến Trúc Tổng Quan

| Aspect | Score | Notes |
|--------|-------|-------|
| **Clean Architecture Compliance** | 8/10 | Good layering, minor violations |
| **SOLID Principles** | 9/10 | Well-implemented |
| **DDD Principles** | 7/10 | Good aggregates, needs more domain services |
| **Code Maintainability** | 8/10 | Clear structure, high volume of boilerplate |
| **Testability** | 8/10 | DI makes testing easy |
| **Performance** | 7/10 | Multiple layers add overhead |
| **Scalability** | 9/10 | Microservices + CQRS + Event-driven |
| **Observability** | 8/10 | Logging, tracing, metrics in place |

### Điểm Mạnh

1. ✅ **Rõ ràng layer separation** - Domain layer zero dependencies
2. ✅ **CQRS pattern** - Clear separation of read/write
3. ✅ **Event-driven architecture** - Loosely coupled services
4. ✅ **Outbox pattern** - Reliable messaging
5. ✅ **Auto-registration** - Scrutor, MediatR reduce boilerplate
6. ✅ **Multiple DB technologies** - Right tool for right job

### Cần Cải Tiến

1. ⚠️ **Giảm boilerplate** - Scaffolding tools, source generators
2. ⚠️ **More domain services** - Move pure business logic to Domain layer
3. ⚠️ **Repository pattern** - Remove direct DB access from handlers
4. ⚠️ **Rich domain models** - More encapsulated behaviors
5. ⚠️ **Read models** - Optimize for queries (CQRS)
6. ⚠️ **Performance monitoring** - Benchmark, profile critical paths

### Kết Luận

Dự án **ProG Coder Shop Microservices** đã thực hiện tốt **Clean Architecture** với các đặc điểm:

- **Architecture pattern chuẩn xác**: Domain → Application → Infrastructure → Presentation
- **SOLID principles tuân thủ tốt**: DIP, SRP, ISP, OCP, LSP
- **Event-driven architecture**: Domain events → Integration events → Outbox → Message bus
- **Polyglot persistence**: Marten, EF Core, MongoDB
- **Modern tech stack**: .NET 8, Carter, MediatR, MassTransit, OpenTelemetry

Tuy nhiên, cần cải thiện:
- Giảm boilerplate code bằng automation
- Tăng cường Domain Model (rich behaviors)
- Đảm bảo 100% tuân thủ Clean Architecture (không có direct DB access)
- Thêm read-optimized models cho CQRS

---

**Document Version:** 1.0.0  
**Last Updated:** January 19, 2026  
**Maintained By:** ProG Coder Team
