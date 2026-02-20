# Phân Tích Decorator Pattern Trong ProgCoder Shop Microservices

## Tóm Tắt

Báo cáo này phân tích chi tiết cách sử dụng **Decorator Pattern** trong dự án ProgCoder Shop Microservices.

---

## 1. Decorator Pattern Là Gì? (Giải thích đơn giản)

### 1.1. Khái niệm dễ hiểu

**Decorator Pattern** là cách bạn **"bọc" một cái gì đó bên ngoài một object khác** để thêm chức năng mới, mà không cần sửa code bên trong object đó.

### 1.2. Ví dụ thực tế

Tưởng tượng bạn có một **chiếc áo thun cơ bản** (đây là object gốc). Bây giờ bạn muốn:

- Thêm logo → bạn **in logo lên áo** (decorator 1)
- Thêm túi → bạn **may thêm túi vào áo** (decorator 2)
- Thêm mũ → bạn **may thêm mũ liền** (decorator 3)

**Điểm quan trọng:** Bạn không cần may lại chiếc áo từ đầu, mà chỉ thêm "lớp" bên ngoài!

### 1.3. Cấu trúc Decorator Pattern

```
┌────────────────────────────────────────────┐
│          Interface chung                   │
│       (IBasketRepository)                  │
│  - GetBasketAsync()                        │
│  - StoreBasketAsync()                      │
│  - DeleteBasketAsync()                     │
└────────────────────────────────────────────┘
                    ▲
                    │ implements
    ┌───────────────┼───────────────┐
    │               │               │
┌───▼────┐    ┌────▼────┐    ┌────▼────┐
│ Basket │    │ Cached  │    │  Other  │
│Repository│   │Basket   │    │Decorator│
│(Gốc)    │   │Repository│   │         │
└────────┘    └────┬────┘    └─────────┘
                   │
                   │ Wraps (bọc bên ngoài)
                   ▼
           ┌───────────────┐
           │ BasketRepository│
           │   (bên trong)   │
           └───────────────┘
```

### 1.4. Nguyên tắc hoạt động

1. Cả **object gốc** và **decorator** đều implement cùng một **interface**
2. Decorator **giữ reference** đến object gốc (gọi là `_repository` hoặc `_inner`)
3. Khi gọi method, decorator có thể:
   - **Trước khi** gọi object gốc: thêm logic (kiểm tra cache, log, validate...)
   - **Gọi** object gốc: `await _repository.GetBasketAsync()`
   - **Sau khi** gọi object gốc: thêm logic (lưu cache, log kết quả...)

### 1.5. Tại sao dùng Decorator Pattern?

| Không dùng Decorator | Dùng Decorator |
|---------------------|----------------|
| Sửa code trong BasketRepository để thêm cache | Tạo CachedBasketRepository, không sửa BasketRepository |
| BasketRepository phải biết về cache | BasketRepository chỉ làm 1 việc: truy cập DB |
| Khó test, khó bảo trì | Dễ test từng phần riêng biệt |
| Không thể tắt cache khi cần | Chỉ cần không đăng ký decorator |

### 1.6. Tóm lại

**Decorator Pattern = "Bọc" object gốc trong một object khác để thêm chức năng**

- Interface giống nhau → code gọi không biết có decorator
- Object gốc không thay đổi → tuân thủ Open/Closed Principle
- Có thể "xếp chồng" nhiều decorator: Logging → Validation → Caching → Handler

## 2. Cac Trien Khai Decorator Pattern Trong Du An

### 2.1. CachedBasketRepository - Repository Decorator

**Vi tri:** `src/Services/Basket/Core/Basket.Infrastructure/Repositories/CachedBasketRepository.cs`

**Mo ta:** Decorator cho Basket Repository them chuc nang caching su dung Redis.

#### Code Implementation

```csharp
public sealed class CachedBasketRepository(
    IBasketRepository repository, 
    IDistributedCache cache) : IBasketRepository
{
    private readonly IBasketRepository _repository = repository;
    private readonly IDistributedCache _cache = cache;

    public async Task<ShoppingCartEntity?> GetBasketAsync(
        string userName, 
        CancellationToken cancellationToken = default)
    {
        // Check cache first
        var cachedBasket = await _cache.GetStringAsync(userName, cancellationToken);

        if (!string.IsNullOrEmpty(cachedBasket))
        {
            return JsonConvert.DeserializeObject<ShoppingCartEntity>(cachedBasket);
        }

        // Cache miss - get from repository
        var basket = await _repository.GetBasketAsync(userName, cancellationToken);

        // Store in cache
        if (basket != null)
        {
            await _cache.SetStringAsync(
                userName,
                JsonConvert.SerializeObject(basket),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(1),
                    SlidingExpiration = TimeSpan.FromHours(1)
                },
                cancellationToken);
        }

        return basket;
    }

    public async Task<ShoppingCartEntity> StoreBasketAsync(
        ShoppingCartEntity basket, 
        CancellationToken cancellationToken = default)
    {
        var result = await _repository.StoreBasketAsync(basket, cancellationToken);

        await _cache.SetStringAsync(
            basket.UserName,
            JsonConvert.SerializeObject(result),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(1),
                SlidingExpiration = TimeSpan.FromHours(1)
            },
            cancellationToken);

        return result;
    }

    public async Task<bool> DeleteBasketAsync(
        string userName, 
        CancellationToken cancellationToken = default)
    {
        var result = await _repository.DeleteBasketAsync(userName, cancellationToken);

        if (result)
        {
            await _cache.RemoveAsync(userName, cancellationToken);
        }

        return result;
    }
}
```

#### Phan Tich Chi Tiet

| Thanh phan | Mo ta |
|------------|-------|
| **Interface** | `IBasketRepository` - dinh nghia cac phuong thuc cua repository |
| **Concrete Component** | `BasketRepository` - implementation co ban |
| **Decorator** | `CachedBasketRepository` - them chuc nang caching |
| **Inner Component** | `_repository` - reference den repository thuc su |
| **Added Behavior** | `_cache` - Redis distributed cache |

#### Cach Hoat Dong

1. **GetBasketAsync:**
   - Kiem tra cache truoc (cache-aside pattern)
   - Neu cache miss, goi `_repository.GetBasketAsync()`
   - Luu ket qua vao cache voi TTL 1 ngay

2. **StoreBasketAsync:**
   - Goi `_repository.StoreBasketAsync()` de luu vao database
   - Dong thoi cap nhat cache (write-through pattern)

3. **DeleteBasketAsync:**
   - Goi `_repository.DeleteBasketAsync()` de xoa tu database
   - Xoa cache entry tuong ung (cache invalidation)

#### Dang Ky Dependency Injection

```csharp
// File: src/Services/Basket/Core/Basket.Infrastructure/DependencyInjection.cs
// Line: 39

services.Decorate<IBasketRepository, CachedBasketRepository>();
```

Su dung Scrutor library de tu dong decorate interface voi decorator class.

---

### 2.2. ValidationBehavior - MediatR Pipeline Decorator

**Vi tri:** `src/Shared/BuildingBlocks/Behaviors/ValidationBehavior.cs`

**Mo ta:** Decorator cho MediatR request handlers them chuc nang validation tu dong.

#### Code Implementation

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>
    (IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);

        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        var failures = validationResults
            .Where(r => r.Errors.Any())
            .SelectMany(r => r.Errors)
            .ToList();

        if (failures.Any()) 
            throw new ValidationException(failures);

        return await next();
    }
}
```

#### Phan Tich Chi Tiet

| Thanh phan | Mo ta |
|------------|-------|
| **Interface** | `IPipelineBehavior<TRequest, TResponse>` - MediatR pipeline interface |
| **Concrete Component** | Command Handlers (CreateProductCommandHandler, v.v.) |
| **Decorator** | `ValidationBehavior` - them validation layer |
| **Inner Component** | `next` delegate - reference den handler tiep theo trong pipeline |
| **Added Behavior** | `validators` - FluentValidation validators |

#### Cach Hoat Dong

1. **Pre-processing:**
   - Tao validation context tu request
   - Chay tat ca validators da dang ky cho request type
   - Thu thap validation errors

2. **Validation Check:**
   - Neu co errors, throw `ValidationException`
   - Request khong den duoc handler chinh

3. **Post-processing:**
   - Neu validation pass, goi `next()` de tiep tuc pipeline
   - Handler chinh duoc thuc thi

---

### 2.3. LoggingBehavior - MediatR Pipeline Decorator

**Vi tri:** `src/Shared/BuildingBlocks/Behaviors/LoggingBehavior.cs`

**Mo ta:** Decorator cho MediatR request handlers them chuc nang logging va performance monitoring.

#### Code Implementation

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>
    (ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull, IRequest<TResponse>
    where TResponse : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        logger.LogInformation(
            "[START] Handle request={Request} - Response={Response} - RequestData={RequestData}",
            typeof(TRequest).Name, typeof(TResponse).Name, request);

        var timer = new Stopwatch();
        timer.Start();

        var response = await next();

        timer.Stop();
        var timeTaken = timer.Elapsed;
        
        if (timeTaken.Seconds > 3)
            logger.LogWarning(
                "[PERFORMANCE] The request {Request} took {TimeTaken} seconds.",
                typeof(TRequest).Name, timeTaken.Seconds);

        logger.LogInformation(
            "[END] Handled {Request} with {Response}",
            typeof(TRequest).Name, typeof(TResponse).Name);
            
        return response;
    }
}
```

#### Phan Tich Chi Tiet

| Thanh phan | Mo ta |
|------------|-------|
| **Interface** | `IPipelineBehavior<TRequest, TResponse>` - MediatR pipeline interface |
| **Concrete Component** | Command/Query Handlers |
| **Decorator** | `LoggingBehavior` - them logging va monitoring |
| **Inner Component** | `next` delegate - reference den handler tiep theo |
| **Added Behavior** | `logger` - ILogger cho logging; `Stopwatch` cho performance tracking |

#### Cach Hoat Dong

1. **Pre-processing:**
   - Log thong tin request bat dau
   - Ghi lai request type, response type, va request data
   - Bat dau do thoi gian thuc thi

2. **Execution:**
   - Goi `next()` de thuc thi handler chinh
   - Do thoi gian thuc thi

3. **Post-processing:**
   - Log thoi gian thuc thi
   - Canh bao neu request mat hon 3 giay
   - Log ket thuc request

---

## 3. So Sanh Cac Loai Decorator

| Tieu chi | CachedBasketRepository | ValidationBehavior | LoggingBehavior |
|----------|------------------------|-------------------|-----------------|
| **Muc dich** | Caching | Validation | Logging/Monitoring |
| **Ap dung cho** | Repository | MediatR Handlers | MediatR Handlers |
| **Thoi diem thuc thi** | Runtime (method calls) | Pre-processing | Pre/Post-processing |
| **Co the chain** | Co (nhieu repository decorators) | Co (pipeline) | Co (pipeline) |
| **Library ho tro** | Scrutor | MediatR | MediatR |

---

## 4. Uu Diem Cua Decorator Pattern Trong Du An

### 4.1. Single Responsibility Principle
- Moi decorator chi co mot nhiem vu duy nhat
- ValidationBehavior chi lam validation
- LoggingBehavior chi lam logging
- CachedBasketRepository chi lam caching

### 4.2. Open/Closed Principle
- Mo rong functionality ma khong sua doi code cu
- Them caching cho repository khong can thay doi BasketRepository
- Them validation khong can thay doi command handlers

### 4.3. Code Reusability
- ValidationBehavior duoc dung cho tat ca commands
- LoggingBehavior duoc dung cho tat ca requests
- Co the ap dung CachedBasketRepository pattern cho cac repository khac

### 4.4. Testability
- Co the test decorator va component rieng biet
- Co the mock inner component trong unit tests
- De dang kiem tra behavior duoc them vao

---

## 5. Cach Dang Ky Decorator Trong DI Container

### 5.1. Repository Decorator (Scrutor)

```csharp
// Dang ky repository co ban
services.AddSingleton<IBasketRepository, BasketRepository>();

// Decorate voi CachedBasketRepository
services.Decorate<IBasketRepository, CachedBasketRepository>();
```

### 5.2. MediatR Pipeline Behaviors

```csharp
// Trong Program.cs hoac DependencyInjection.cs
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<Program>();
    
    // Dang ky behaviors theo thu tu
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
});
```

**Thu tu thuc thi pipeline:**
1. LoggingBehavior (Start)
2. ValidationBehavior
3. Handler thuc su
4. ValidationBehavior (End)
5. LoggingBehavior (End)

---

## 6. Ket Luan

Decorator Pattern duoc su dung hieu qua trong ProgCoder Shop Microservices de:

1. **Them cross-cutting concerns** (logging, validation, caching) ma khong lam phuc tap business logic
2. **Tach biet concerns** - moi decorator co mot nhiem vu ro rang
3. **Tang tinh linh hoat** - de dang them/bot functionality
4. **Tuan thu SOLID principles** - dac biet la Single Responsibility va Open/Closed

Cac trien khai chinh:
- **CachedBasketRepository**: Caching layer cho repository
- **ValidationBehavior**: Tu dong validation cho MediatR requests
- **LoggingBehavior**: Logging va performance monitoring

---

## 7. Tai Lieu Tham Khao

- **Files phan tich:**
  - `src/Services/Basket/Core/Basket.Infrastructure/Repositories/CachedBasketRepository.cs`
  - `src/Services/Basket/Core/Basket.Infrastructure/DependencyInjection.cs`
  - `src/Shared/BuildingBlocks/Behaviors/ValidationBehavior.cs`
  - `src/Shared/BuildingBlocks/Behaviors/LoggingBehavior.cs`

- **Libraries su dung:**
  - Scrutor (DI decoration)
  - MediatR (Pipeline behaviors)
  - FluentValidation (Validation)

---

*Bao cao duoc tao bang cach su dung repomix va ast-grep skills de phan tich codebase.*
