# Phân Tích Decorator Pattern Trong ProgCoder Shop Microservices

## Tóm Tắt

Báo cáo này phân tích cách sử dụng Decorator Pattern trong dự án ProgCoder Shop Microservices. Decorator Pattern được áp dụng thông qua nhiều cơ chế khác nhau trong .NET ecosystem.

## Các Cơ Chế Decorator Pattern Được Sử Dụng

### 1. EF Core Interceptors (Database Decorators)

**Vị trí:** `src/Services/Order/Core/Order.Infrastructure/Data/Interceptors/`

#### 1.1 AuditableEntityInterceptor

**File:** `AuditableEntityInterceptor.cs`

```csharp
public sealed class AuditableEntityInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(DbContextEventData eventData, InterceptionResult<int> result)
    {
        UpdateEntities(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(...)
    {
        UpdateEntities(eventData.Context);
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

**Mô tả:**
- Kế thừa từ `SaveChangesInterceptor` của EF Core
- Decorator cho phương thức `SaveChanges` và `SaveChangesAsync`
- Tự động cập nhật `CreatedOnUtc` và `LastModifiedOnUtc` cho các entity implement `IAuditable`
- Gọi `base.SavingChanges()` để tiếp tục chuỗi xử lý (chain of responsibility)

**Pattern:** Decorator Pattern - Wrap và mở rộng behavior của `SaveChanges` mà không thay đổi core logic.

#### 1.2 DispatchDomainEventsInterceptor

**File:** `DispatchDomainEventsInterceptor.cs`

```csharp
public class DispatchDomainEventsInterceptor(IMediator mediator) : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(DbContextEventData eventData, InterceptionResult<int> result)
    {
        DispatchDomainEvents(eventData.Context).GetAwaiter().GetResult();
        return base.SavingChanges(eventData, result);
    }

    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(...)
    {
        await DispatchDomainEvents(eventData.Context);
        return await base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

**Mô tả:**
- Decorator cho `SaveChanges` operations
- Tự động dispatch domain events trước khi lưu vào database
- Sử dụng MediatR để publish events
- Tuân thủ nguyên tắc: gọi `base` method để đảm bảo chain không bị đứt

**Pattern:** Decorator Pattern + Chain of Responsibility

---

### 2. gRPC Interceptors

#### 2.1 GrpcApiKeyInterceptor (Client-side)

**File:** `src/Services/Basket/Core/Basket.Infrastructure/GrpcClients/Interceptors/GrpcApiKeyInterceptor.cs`

```csharp
public sealed class GrpcApiKeyInterceptor(IConfiguration cfg) : Interceptor
{
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var grpcKey = cfg.GetValue<string>(...);
        var headers = context.Options.Headers ?? [];
        if (!headers.Any(h => h.Key.Equals(ReqHeaderName.GrpcKey, ...)))
        {
            headers.Add(ReqHeaderName.GrpcKey, grpcKey);
        }

        var newOptions = context.Options.WithHeaders(headers);
        var newContext = new ClientInterceptorContext<TRequest, TResponse>(
            context.Method, context.Host, newOptions);

        return base.AsyncUnaryCall(request, newContext, continuation);
    }
}
```

**Mô tả:**
- Kế thừa từ `Interceptor` của gRPC
- Decorator cho gRPC client calls
- Tự động thêm API key vào headers của mọi gRPC request
- Tạo context mới với headers đã được enrich

**Pattern:** Decorator Pattern - Enrich request mà không thay đổi service implementation.

#### 2.2 ApiKeyValidationInterceptor (Server-side)

**File:** `src/Services/Order/Api/Order.Grpc/Interceptors/ApiKeyValidationInterceptor.cs`

```csharp
public class ApiKeyValidationInterceptor(IConfiguration cfg) : Interceptor
{
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var provided = context.RequestHeaders.FirstOrDefault(h => h.Key == ReqHeaderName.GrpcKey)?.Value;
        var grpcKey = cfg.GetValue<string>(...);

        if (string.IsNullOrEmpty(provided) || !TimeConstantEquals(provided, grpcKey!))
        {
            throw new RpcException(new Status(StatusCode.Unauthenticated, MessageCode.Unauthorized));
        }

        return await continuation(request, context);
    }
}
```

**Mô tả:**
- Server-side gRPC interceptor
- Decorator cho UnaryServerHandler
- Validate API key trước khi cho phép request đi vào handler
- Sử dụng constant-time comparison để tránh timing attacks

**Pattern:** Decorator Pattern - Cross-cutting concern (security) được tách ra khỏi business logic.

---

### 3. MediatR Pipeline Behaviors

**Vị trí:** `src/Shared/BuildingBlocks/Behaviors/`

#### 3.1 ValidationBehavior

**File:** `ValidationBehavior.cs`

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>
    (IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);
        var validationResults = await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, cancellationToken)));
        var failures = validationResults.Where(r => r.Errors.Any()).SelectMany(r => r.Errors).ToList();

        if (failures.Any()) throw new ValidationException(failures);

        return await next();
    }
}
```

**Mô tả:**
- Implement `IPipelineBehavior<TRequest, TResponse>` của MediatR
- Decorator cho command handlers
- Thực hiện validation trước khi handler thực thi
- Nếu validation fail, throw exception và không gọi `next()`
- Nếu validation pass, gọi `next()` để tiếp tục pipeline

**Pattern:** Decorator Pattern + Pipeline Pattern

#### 3.2 LoggingBehavior

**File:** `LoggingBehavior.cs`

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>
    (ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull, IRequest<TResponse>
    where TResponse : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        logger.LogInformation("[START] Handle request={Request} - Response={Response} - RequestData={RequestData}", ...);
        
        var timer = new Stopwatch();
        timer.Start();
        
        var response = await next();
        
        timer.Stop();
        var timeTaken = timer.Elapsed;
        if (timeTaken.Seconds > 3)
            logger.LogWarning("[PERFORMANCE] The request {Request} took {TimeTaken} seconds.", ...);
        
        logger.LogInformation("[END] Handled {Request} with {Response}", ...);
        return response;
    }
}
```

**Mô tả:**
- Decorator cho tất cả MediatR requests
- Log start/end của mỗi request
- Performance monitoring - cảnh báo nếu request > 3 giây
- Cross-cutting concern (logging) được tách biệt khỏi business logic

**Pattern:** Decorator Pattern + AOP (Aspect-Oriented Programming)

---

## Cấu Trúc Decorator Pattern Trong Dự Án

### Class Diagram (Conceptual)

```
┌─────────────────────────────────────────────────────────────────┐
│                    DECORATOR PATTERN USAGE                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│   SaveChanges       │◄────│ AuditableEntity     │
│   (EF Core)         │     │   Interceptor       │
└─────────────────────┘     └─────────────────────┘
                                    │
                                    ▼
                           ┌─────────────────────┐
                           │ DispatchDomainEvents│
                           │   Interceptor       │
                           └─────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│   gRPC Call         │◄────│ GrpcApiKey          │
│   (Client)          │     │   Interceptor       │
└─────────────────────┘     └─────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│   gRPC Handler      │◄────│ ApiKeyValidation    │
│   (Server)          │     │   Interceptor       │
└─────────────────────┘     └─────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│   Command Handler   │◄────│ ValidationBehavior  │
│   (MediatR)         │     │   (Pipeline)        │
└─────────────────────┘     └─────────────────────┘
                                    │
                                    ▼
                           ┌─────────────────────┐
                           │ LoggingBehavior     │
                           │   (Pipeline)        │
                           └─────────────────────┘
```

---

## Đăng Ký Decorators (Dependency Injection)

### EF Core Interceptors

```csharp
// Trong DbContext configuration
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.AddInterceptors(
        new AuditableEntityInterceptor(),
        new DispatchDomainEventsInterceptor(mediator)
    );
}
```

### MediatR Pipeline Behaviors

```csharp
// Trong Program.cs hoặc DI configuration
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<Program>();
});

// Behaviors được đăng ký tự động hoặc:
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
```

### gRPC Interceptors

```csharp
// Client-side
var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    Interceptors = { new GrpcApiKeyInterceptor(configuration) }
});

// Server-side
services.AddGrpc(options =>
{
    options.Interceptors.Add<ApiKeyValidationInterceptor>();
});
```

---

## Lợi Ích Của Việc Sử Dụng Decorator Pattern

### 1. Single Responsibility Principle (SRP)
- Mỗi decorator chỉ chịu trách nhiệm cho một cross-cutting concern
- Business logic không bị lẫn với logging, validation, security

### 2. Open/Closed Principle (OCP)
- Có thể thêm behavior mới mà không cần sửa đổi code hiện có
- Ví dụ: Thêm caching behavior mà không thay đổi handlers

### 3. Composability
- Có thể kết hợp nhiều decorators với nhau
- Thứ tự execution được kiểm soát qua registration order

### 4. Testability
- Mỗi decorator có thể được unit test độc lập
- Dễ dàng mock dependencies

### 5. Reusability
- Cùng một decorator có thể áp dụng cho nhiều services/commands

---

## Thứ Tự Thực Thi (Execution Order)

Khi có nhiều decorators, thứ tự thực thi tuân theo **Decorator Stack Pattern**:

```
Request Flow:
ValidationBehavior → LoggingBehavior → Actual Handler
     │                    │                  │
     ▼                    ▼                  ▼
  Validate()          Log Start         Execute
     │                    │                  │
     ▼                    ▼                  ▼
  next()              next()            Return
     │                    │                  │
     └────────────────────┴──────────────────┘
                          │
                    Response Flow:
                    Return Result
                          │
                    Log End (LoggingBehavior)
                          │
                    Return Result
                          │
                    Return Result (ValidationBehavior)
```

---

## Các Frameworks Hỗ Trợ Decorator Pattern

| Framework | Interface/Class | Pattern Type |
|-----------|----------------|--------------|
| EF Core | `IInterceptor`, `SaveChangesInterceptor` | Database Interceptor |
| gRPC | `Interceptor` | RPC Interceptor |
| MediatR | `IPipelineBehavior<TRequest, TResponse>` | Pipeline Decorator |
| ASP.NET Core | `IActionFilter`, `IExceptionFilter` | HTTP Filter |

---

## Kết Luận

Dự án ProgCoder Shop Microservices sử dụng Decorator Pattern một cách hiệu quả thông qua:

1. **EF Core Interceptors** - Xử lý cross-cutting concerns ở data layer (auditing, domain events)
2. **gRPC Interceptors** - Xử lý security và enrichment ở service communication layer
3. **MediatR Pipeline Behaviors** - Xử lý validation, logging, và các concerns khác ở application layer

Cách tiếp cận này giúp:
- Tách biệt concerns rõ ràng
- Dễ dàng maintain và extend
- Tuân thủ SOLID principles
- Code sạch và dễ test

---

## Tài Liệu Tham Khảo

- **Các file phân tích:**
  - `src/Services/Order/Core/Order.Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs`
  - `src/Services/Order/Core/Order.Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs`
  - `src/Services/Basket/Core/Basket.Infrastructure/GrpcClients/Interceptors/GrpcApiKeyInterceptor.cs`
  - `src/Services/Order/Api/Order.Grpc/Interceptors/ApiKeyValidationInterceptor.cs`
  - `src/Shared/BuildingBlocks/Behaviors/ValidationBehavior.cs`
  - `src/Shared/BuildingBlocks/Behaviors/LoggingBehavior.cs`

- **Tools sử dụng:**
  - Repomix - Pack codebase để phân tích
  - AST-Grep - Tìm kiếm pattern theo cấu trúc AST
  - Glob/Grep - Tìm kiếm file và pattern

---

*Ngày phân tích: 2025-02-01*
*Dự án: ProgCoder Shop Microservices*
