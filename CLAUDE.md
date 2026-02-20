# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ProgCoder Shop Microservices** is a .NET 8 e-commerce microservices architecture implementing:
- **Clean Architecture** with Domain-Driven Design (DDD)
- **CQRS pattern** with MediatR for command/query separation
- **Event-driven architecture** using MassTransit/RabbitMQ with Outbox/Inbox patterns
- **Multiple databases**: PostgreSQL, MySQL, SQL Server, MongoDB, Marten (PostgreSQL), Redis, Elasticsearch
- **gRPC** for inter-service communication
- **React/Next.js** frontends (App.Admin, App.Store)
- **YARP** API Gateway for unified routing

## Development Commands

### Docker Compose (All Services)

```bash
# Build and start all services
docker-compose up --build -d

# Start infrastructure only (for development mode)
docker-compose -f docker-compose.infrastructure.yml up -d

# View logs
docker-compose logs -f [service-name]

# Stop services
docker-compose down

# Stop with volumes (clean slate)
docker-compose down -v
```

### Database Migrations

```bash
# Apply existing migrations (Linux/Mac/WSL)
chmod +x run-migration-linux.sh
./run-migration-linux.sh

# Create new migrations
chmod +x add-migration-linux.sh
./add-migration-linux.sh
```

### Running Individual Services

```bash
# Backend API (example: Catalog Service)
cd src/Services/Catalog/Api/Catalog.Api
dotnet run

# Worker Service (example: Catalog Outbox Worker)
cd src/Services/Catalog/Worker/Catalog.Worker.Outbox
dotnet run

# Frontend (App.Admin)
cd src/Apps/App.Admin
npm install
npm run dev
```

### Tests

```bash
# Run all tests
dotnet test

# Run specific service tests
cd src/Services/[ServiceName]/Tests
dotnet test
```

### Adding EF Core Migration

```bash
cd src/Services/[ServiceName]/Infrastructure
dotnet ef migrations add [MigrationName] -s ../Api/[ServiceName].Api
```

## Solution Structure

```
src/
├── Services/                    # Microservices (Clean Architecture)
│   └── {ServiceName}/
│       ├── Api/                 # Carter endpoints, Program.cs
│       ├── Core/
│       │   ├── {Service}.Domain/        # Entities, Value Objects, Domain Events
│       │   ├── {Service}.Application/   # CQRS Commands/Queries, Handlers, DTOs
│       │   └── {Service}.Infrastructure/ # Repositories, EF Core, gRPC clients
│       └── Worker/              # Background jobs (Outbox, Consumers)
├── Shared/
│   ├── BuildingBlocks/          # CQRS, Behaviors, Exceptions, Pagination
│   ├── Common/                  # Constants, Models, Extensions, Helpers
│   ├── Contracts/               # gRPC Protos
│   └── EventSourcing/           # Integration Events
├── ApiGateway/                  # YARP reverse proxy
├── JobOrchestrator/             # Quartz.NET scheduled jobs
└── Apps/                        # Next.js frontends (App.Admin, App.Store)
```

## Architecture Patterns

### Clean Architecture Layers

1. **Domain Layer** (innermost) - Entities, Aggregates, Value Objects, Domain Events, Domain Exceptions. No external dependencies.
2. **Application Layer** - CQRS Commands/Queries, Handlers, DTOs, Repository Interfaces, Validators, Mappings. Depends only on Domain.
3. **Infrastructure Layer** - EF Core repositories, External services, gRPC/HTTP clients. Implements Application interfaces.
4. **API Layer** - Carter minimal API endpoints. Entry point only.

### CQRS with MediatR

**Commands** (write operations):
```csharp
public sealed record CreateProductCommand(CreateProductDto Dto, Actor Actor) : ICommand<Guid>;

public sealed class CreateProductCommandHandler(
    IMapper mapper,
    IProductRepository repository) : ICommandHandler<CreateProductCommand, Guid>
{
    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken cancellationToken)
    {
        // Handler logic
    }
}
```

**Queries** (read operations):
```csharp
public sealed record GetProductQuery(Guid Id) : IQuery<GetProductResult>;

public sealed class GetProductResult
{
    public ProductDto Item { get; init; }
    public GetProductResult(ProductDto item) { Item = item; }
}

public sealed class GetProductQueryHandler(
    IProductRepository repository,
    IMapper mapper) : IQueryHandler<GetProductQuery, GetProductResult>
{
    public async Task<GetProductResult> Handle(GetProductQuery query, CancellationToken cancellationToken)
    {
        var product = await repository.GetByIdAsync(query.Id, cancellationToken)
            ?? throw new NotFoundException(MessageCode.ProductNotFound);
        var dto = mapper.Map<ProductDto>(product);
        return new GetProductResult(dto);
    }
}
```

### Repository & Unit of Work Pattern

**CRITICAL Rules**:
- NEVER use `IApplicationDbContext` directly in Application layer
- ALWAYS use `IUnitOfWork` to access repositories
- Repository methods with `Include` MUST have `WithRelationship` suffix
- Read-only queries MUST use `AsNoTracking()`

```csharp
// Command Handler using UnitOfWork
public sealed class CreateInventoryItemCommandHandler(
    IUnitOfWork unitOfWork,
    ICatalogGrpcService catalogGrpc) : ICommandHandler<CreateInventoryItemCommand, Guid>
{
    public async Task<Guid> Handle(CreateInventoryItemCommand command, CancellationToken ct)
    {
        var product = await catalogGrpc.GetProductByIdAsync(dto.ProductId.ToString(), ct)
            ?? throw new ClientValidationException(MessageCode.ProductIsNotExists, dto.ProductId);

        var entity = InventoryItemEntity.Create(...);
        await unitOfWork.InventoryItems.AddAsync(entity, ct);
        await unitOfWork.SaveChangesAsync(ct);
        return entity.Id;
    }
}
```

### Outbox/Inbox Pattern for Reliable Messaging

1. **Outbox**: Domain events → Integration events stored in Outbox table → Worker publishes to RabbitMQ
2. **Inbox**: Consume messages → Store in Inbox table (idempotency) → Process → Mark as processed

## Important Code Conventions

### MessageCode Convention

**NEVER use direct string messages** - always use `MessageCode` constants from `src/Shared/Common/Constants/MessageCode.cs`:

```csharp
// CORRECT
throw new NotFoundException(MessageCode.ProductNotFound);
RuleFor(x => x.Dto.Name).NotEmpty().WithMessage(MessageCode.ProductNameIsRequired);

// WRONG
throw new NotFoundException("Product not found");
RuleFor(x => x.Dto.Name).NotEmpty().WithMessage("Name is required");
```

### Exception Handling

**In Application Layer, use ONLY these exceptions**:
- `ClientValidationException` - Client input validation errors (400)
- `NotFoundException` - Resource not found (404)
- `DomainException` - Domain rule violations (thrown from Domain layer)

**NEVER create custom exceptions** or use `BadRequestException`, `ArgumentException`, etc.

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Entity | XxxEntity | ProductEntity |
| DTO | XxxDto | CreateProductDto |
| Command | XxxCommand | CreateProductCommand |
| Query | XxxQuery | GetProductQuery |
| Result | XxxResult | GetProductResult |
| Repository | IXxxRepository | IProductRepository |
| Handler | XxxCommandHandler/QueryHandler | CreateProductCommandHandler |

### Primary Constructor Pattern (Preferred)

```csharp
public sealed class CreateProductCommandHandler(
    IProductRepository repository,
    IMapper mapper,
    ILogger<CreateProductCommandHandler> logger) : ICommandHandler<CreateProductCommand, Guid>
{
    #region Implementations

    public async Task<Guid> Handle(CreateProductCommand command, CancellationToken cancellationToken)
    {
        // Implementation
    }

    #endregion
}
```

### Regions Organization

```csharp
public sealed class MyClass
{
    #region Fields, Properties and Indexers
    #endregion

    #region Ctors
    #endregion

    #region Implementations
    #endregion

    #region Factories
    #endregion

    #region Methods
    #endregion

    #region Private Methods
    #endregion
}
```

## Frontend Conventions (Next.js)

- Use **Formik + Yup** for all forms
- **ALL text MUST use i18n** with `t("key")` - NO fallback patterns
- API calls created as services in `services/` directory
- Check `/api/endpoints.js` before creating new API calls
- In catch blocks: **ONLY use `console.error()`** - NO toast notifications

## Service Access URLs

- **API Gateway**: http://localhost:15009
- **App.Admin**: http://localhost:3001
- **App.Store**: http://localhost:3002
- **Keycloak**: http://localhost:8080 (admin/admin)
- **RabbitMQ Management**: http://localhost:15673
- **Grafana**: http://localhost:3000

## API Gateway Routes

| Route | Service |
|-------|---------|
| `/catalog-service/**` | Catalog (5001) |
| `/basket-service/**` | Basket (5006) |
| `/order-service/**` | Order (5005) |
| `/inventory-service/**` | Inventory (5002) |
| `/discount-service/**` | Discount (5004) |
| `/notification-service/**` | Notification (5003) |
| `/report-service/**` | Report (5007) |
| `/search-service/**` | Search (5008) |
| `/communication-service/**` | Communication (5009) |

## Default Credentials

| Service | Username | Password |
|---------|----------|----------|
| Keycloak | admin | admin |
| Grafana | admin | admin |
| RabbitMQ | admin | 123456789Aa |
| PostgreSQL | postgres | 123456789Aa |
| MongoDB | mongodb | 123456789Aa |
| MySQL | root | 123456789Aa |
| SQL Server | sa | 123456789Aa |
| MinIO | minioadmin | minioadmin |
| Elasticsearch | elastic | elastic123 |
| Redis | - | 123456789Aa |

## Key Dependencies

- **MediatR** 12.3.0 - CQRS pattern
- **FluentValidation** 11.9.2 - Input validation
- **AutoMapper** 12.0.1 - Object mapping
- **MassTransit.RabbitMQ** 8.5.1 - Message broker
- **Carter** 8.1.0 - Minimal API framework
- **Entity Framework Core** 8.0.6 - ORM (SQL Server, PostgreSQL, MySQL)
- **Marten** 7.26.4 - PostgreSQL document store
- **MongoDB.Driver** 3.4.2 - MongoDB driver
- **Scrutor** 6.1.0 - Dependency injection scanning
