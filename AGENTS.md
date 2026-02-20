# AGENTS.md

This file provides guidance for agentic coding assistants working on the ProgCoder Shop Microservices repository.

## Essential Commands

### Backend (.NET)
```bash
# Run tests (all)
dotnet test

# Run specific test class
dotnet test --filter "FullyQualifiedName~ClassName"

# Run specific test method
dotnet test --filter "FullyQualifiedName~MethodName"

# Run tests for specific service
dotnet test src/Services/[ServiceName]/Tests

# Run specific API service
cd src/Services/[ServiceName]/Api/[ServiceName].Api
dotnet run

# Create EF Core migration
cd src/Services/[ServiceName]/Infrastructure
dotnet ef migrations add [Name] -s ../Api/[ServiceName].Api

# Run migrations
./run-migration-linux.sh
```

### Frontend (Next.js)
```bash
cd src/Apps/[App.Admin|App.Store]
npm install
npm run dev
npm run build
npm run lint
```

### Docker
```bash
# All services
docker-compose up --build -d
docker-compose logs -f [service]
docker-compose down

# Infrastructure only
docker-compose -f docker-compose.infrastructure.yml up -d
```

## Code Style - .NET (C#)

### File Organization
- **Using statements**: Inside `#region using` at top, sorted alphabetically
- **Region sections**: 
  ```
  #region Fields, Properties and Indexers
  #region Ctors
  #region Implementations
  #region Factories
  #region Methods
  #region Private Methods
  ```

### Constructors
- **Preferred**: Primary constructor pattern for handlers/services
  ```csharp
  public sealed class CreateProductCommandHandler(
      IProductRepository repository,
      IMapper mapper) : ICommandHandler<CreateProductCommand, Guid>
  ```

### Naming Conventions
- Entities: `XxxEntity` (e.g., `ProductEntity`)
- DTOs: `XxxDto` (e.g., `CreateProductDto`)
- Commands: `XxxCommand` (e.g., `CreateProductCommand`)
- Queries: `XxxQuery` (e.g., `GetProductQuery`)
- Results: `XxxResult` (e.g., `GetProductResult`)
- Repositories: `IXxxRepository` (e.g., `IProductRepository`)
- Handlers: `XxxCommandHandler` or `XxxQueryHandler`

### Exception Handling
**CRITICAL**: In Application Layer, use ONLY these exceptions:
- `ClientValidationException` - Client input validation errors (400)
- `NotFoundException` - Resource not found (404)
- `DomainException` - Domain rule violations (from Domain layer)

NEVER create custom exceptions or use `BadRequestException`, `ArgumentException`, etc.

### Error Messages
**NEVER use direct string messages** - always use `MessageCode` constants:
```csharp
// CORRECT
throw new NotFoundException(MessageCode.ProductNotFound);
RuleFor(x => x.Dto.Name).NotEmpty().WithMessage(MessageCode.ProductNameIsRequired);

// WRONG
throw new NotFoundException("Product not found");
```

### Architecture Rules
- **NEVER use `IApplicationDbContext` directly in Application layer**
- **ALWAYS use `IUnitOfWork` to access repositories**
- Repository methods with `Include` MUST have `WithRelationship` suffix
- Read-only queries MUST use `AsNoTracking()`
- Domain layer: NO external dependencies (EF Core, MediatR, etc.)

### Code Style - Frontend (Next.js/React)

### Forms & Validation
- **ALL forms MUST use Formik + Yup** - no exceptions
- **ALL text MUST use i18n** with `t("key")` - NO fallback patterns

### Error Handling
- In catch blocks: **ONLY use `console.error()`** - NO toast notifications

### API Calls
- Create services in `services/` directory
- Check `/api/endpoints.js` before creating new API calls

## Architecture Patterns

### CQRS Pattern
```csharp
// Command (write)
public sealed record CreateProductCommand(CreateProductDto Dto, Actor Actor) : ICommand<Guid>;

// Query (read)
public sealed record GetProductQuery(Guid Id) : IQuery<GetProductResult>;
public sealed record GetProductResult(ProductDto Item);

// Handler (primary constructor)
public sealed class GetProductQueryHandler(
    IProductRepository repository) : IQueryHandler<GetProductQuery, GetProductResult>
{
    public async Task<GetProductResult> Handle(GetProductQuery query, CancellationToken ct) { ... }
}
```

### Repository Pattern
Use `IUnitOfWork` to access repositories:
```csharp
public async Task<Guid> Handle(CreateProductCommand command, CancellationToken ct)
{
    await unitOfWork.Products.AddAsync(entity, ct);
    await unitOfWork.SaveChangesAsync(ct);
}
```

## Testing

When tests fail:
- **If test is incorrect**: Modify the test
- **If code has logic error**: STOP and inform user - do not fix unilaterally

## Important Notes

- Project is .NET 8 with Clean Architecture + DDD
- Event-driven with MassTransit/RabbitMQ (Outbox/Inbox pattern)
- Multiple DBs: PostgreSQL, MySQL, SQL Server, MongoDB, Marten, Redis, Elasticsearch
- Services use Carter for minimal APIs, gRPC for inter-service
- API Gateway at http://localhost:15009 with YARP
- App.Admin: http://localhost:3001 | App.Store: http://localhost:3002

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd sync
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
