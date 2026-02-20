# Phân tích Hệ thống API Gateway - ProgCoder Shop Microservices

## 1. Tổng quan

API Gateway của dự án ProgCoder Shop Microservices được xây dựng trên nền tảng **YARP (Yet Another Reverse Proxy)** - một thư viện reverse proxy hiệu năng cao được phát triển bởi Microsoft. YARP được thiết kế đặc biệt cho các ứng dụng ASP.NET Core cần khả năng reverse proxy linh hoạt và có thể tùy chỉnh cao.

Trong kiến trúc microservices, API Gateway đóng vai trò là điểm đầu vào duy nhất (Single Entry Point) cho tất cả các request từ phía client. Nó thực hiện việc định tuyến (routing) các request đến các microservice tương ứng, đồng thời tập trung các cross-cutting concerns như authentication, authorization, rate limiting, load balancing, logging và monitoring.

---

## 2. Cấu trúc dự án

### 2.1 Vị trí và tổ chức file

```
src/
└── ApiGateway/
    └── YarpApiGateway/
        ├── Program.cs
        ├── YarpApiGateway.csproj
        ├── appsettings.json
        ├── appsettings.Development.json
        ├── Dockerfile
        └── Properties/
            └── launchSettings.json
```

### 2.2 Project Dependencies

Project sử dụng hai NuGet packages chính:

| Package | Mục đích |
|---------|----------|
| `Yarp.ReverseProxy` | Thư viện reverse proxy chính |
| `AspNetCore.HealthChecks.UI.Client` | Health checks và UI client |

Yarp.ReverseProxy là thành phần cốt lõi, cung cấp khả năng load balancing, routing, transforms và session affinity. Package thứ hai tích hợp hệ thống health checks, cho phép theo dõi trạng thái của API Gateway và các downstream services.

---

## 3. Cấu hình Routing

### 3.1 Route Configuration

API Gateway định tuyến request đến 9 microservices thông qua các route được định nghĩa trong file `appsettings.json`. Mỗi route bao gồm tên route, cluster destination và path pattern tương ứng.

| Route Name | Cluster ID | Path Pattern | Destination Service |
|------------|------------|--------------|-------------------|
| basket-route | basket-cluster | `/basket-service/{**catch-all}` | basket-api:8080 |
| catalog-route | catalog-cluster | `/catalog-service/{**catch-all}` | catalog-api:8080 |
| discount-route | discount-cluster | `/discount-service/{**catch-all}` | discount-api:8080 |
| inventory-route | inventory-cluster | `/inventory-service/{**catch-all}` | inventory-api:8080 |
| notification-route | notification-cluster | `/notification-service/{**catch-all}` | notification-api:8080 |
| order-route | order-cluster | `/order-service/{**catch-all}` | order-api:8080 |
| report-route | report-cluster | `/report-service/{**catch-all}` | report-api:8080 |
| search-route | search-cluster | `/search-service/{**catch-all}` | search-api:8080 |
| communication-route | communication-cluster | `/communication-service/{**catch-all}` | communication-api:8080 |

### 3.2 Route Transformation

Mỗi route sử dụng transform `PathPattern` với cú pháp `{**catch-all}`. Đây là wildcard pattern cho phép chuyển tiếp toàn bộ request path (bao gồm cả query strings) đến downstream service. Ví dụ, request `GET /basket-service/api/v1/baskets/123` sẽ được chuyển đến `http://basket-api:8080/api/v1/baskets/123`.

Transform này đảm bảo rằng cấu trúc URL từ client được giữ nguyên khi request đến backend services, giúp maintain RESTful API contract.

### 3.3 Cluster Configuration

Mỗi service được cấu hình như một cluster với các destinations. Hiện tại, mỗi cluster chỉ có một destination duy nhất:

```json
"catalog-cluster": {
  "Destinations": {
    "destination1": {
      "Address": "http://catalog-api:8080"
    }
  }
}
```

**Lưu ý**: Cấu hình hiện tại chưa có load balancing giữa nhiều instances. Để đảm bảo high availability trong production, nên cấu hình nhiều destinations per cluster hoặc sử dụng service discovery mechanism.

---

## 4. Các tính năng đã triển khai

### 4.1 CORS (Cross-Origin Resource Sharing)

API Gateway được cấu hình cho phép Cross-Origin requests từ các frontend applications:

```json
"CorsConfig": {
  "PolicyName": "AllowSpecificOrigins",
  "Domains": [
    "http://localhost:3000",
    "http://localhost:3001",
    "http://localhost:3002"
  ]
}
```

Chính sách CORS này cho phép:
- Tất cả các HTTP methods (GET, POST, PUT, DELETE, etc.)
- Tất cả các headers
- Credentials (cookies, authorization headers)

Việc cho phép credentials là bắt buộc để SignalR có thể hoạt động đúng, cho phép gửi cookies và tokens trong cross-origin requests.

**Phân tích**: Port 3000, 3001, 3002 tương ứng với:
- Port 3000: Development server chung
- Port 3001: App.Admin (Next.js admin panel)
- Port 3002: App.Store (Next.js storefront)

### 4.2 Rate Limiting

Hệ thống rate limiting được triển khai với chiến lược Fixed Window:

```csharp
builder.Services.AddRateLimiter(rateLimiterOptions =>
{
    rateLimiterOptions.AddFixedWindowLimiter("fixed", options =>
    {
        options.Window = TimeSpan.FromSeconds(10);
        options.PermitLimit = 5;
    });
});
```

**Thông số cấu hình**:
- **Strategy**: Fixed Window (cố định theo khoảng thời gian)
- **Window**: 10 giây
- **Permit Limit**: 5 requests per window

**Đánh giá**: Cấu hình rate limiting hiện tại khá strict với chỉ 5 requests/10 giây. Đây là mức phù hợp cho development nhưng cần điều chỉnh cho production dựa trên:
- Expected traffic patterns
- Business requirements
- Client behavior (polling, real-time updates)

**Khuyến nghị**: Để support distributed deployments, nên migrate sang Redis-based rate limiting với package `AspNetCoreRateLimit.Redis`.

### 4.3 Health Checks

Health check endpoint được triển khai tại `/health` với response format tương thích HealthChecks.UI:

```csharp
app.UseHealthChecks("/health",
    new HealthCheckOptions
    {
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
```

Health check này:
- Kiểm tra trạng thái của API Gateway
- Có thể mở rộng để include health checks của downstream services
- Tích hợp được với Kubernetes liveness/readiness probes
- Cung cấp JSON response cho monitoring systems

### 4.4 Root Endpoint

Endpoint gốc `/` trả về thông tin trạng thái:

```json
{
  "service": "API Gateway",
  "status": "Running",
  "timestamp": "2026-02-20T...",
  "environment": "Development",
  "endpoints": {
    "health": "/health"
  },
  "message": "API Gateway is running..."
}
```

---

## 5. Container Deployment

### 5.1 Docker Configuration

API Gateway được deploy dưới dạng Docker container với cấu hình trong `docker-compose.yml`:

```yaml
api-gateway:
  image: ${DOCKERHUB_USERNAME}/api-gateway:latest
  container_name: api-gateway
  restart: unless-stopped
  ports:
    - ${API_GATEWAY_PORT}:8080
  depends_on:
    - catalog-api
    - basket-api
    - inventory-api
    - order-api
    - discount-api
    - notification-api
    - search-api
    - report-api
    - communication-api
  networks:
    - progcoder_network
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 60s
```

### 5.2 Thông số triển khai

| Thông số | Giá trị |
|----------|---------|
| Container Name | api-gateway |
| Exposed Port | 8080 (ánh xạ qua API_GATEWAY_PORT env) |
| Restart Policy | unless-stopped |
| Health Check Interval | 30s |
| Health Check Timeout | 10s |
| Health Check Retries | 3 |
| Start Period | 60s |

### 5.3 Network Architecture

API Gateway nằm trong network `progcoder_network`, cho phép giao tiếp với tất cả các microservices backend thông qua Docker DNS. Tất cả các service đều sử dụng internal DNS names (ví dụ: `catalog-api`, `basket-api`) thay vì localhost.

---

## 6. Program.cs Analysis

File `Program.cs` chứa toàn bộ logic khởi tạo và cấu hình API Gateway:

```csharp
// 1. CORS Configuration
builder.Services.AddCors(options =>
{
    options.AddPolicy(policyName, b => b
        .WithOrigins(...)
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials());
});

// 2. YARP Reverse Proxy
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// 3. Rate Limiting
builder.Services.AddRateLimiter(...);

// 4. Health Checks
builder.Services.AddHealthChecks();

// Middleware Pipeline
app.UseCors(policyName);
app.UseRateLimiter();
app.MapReverseProxy();
app.UseHealthChecks("/health", ...);
```

**Middleware Order**: Thứ tự middleware rất quan trọng:
1. `UseCors` - Xử lý CORS trước
2. `UseRateLimiter` - Áp dụng rate limiting
3. `MapReverseProxy` - Route request đến downstream
4. `UseHealthChecks` - Health check endpoint

---

## 7. Đánh giá và Khuyến nghị

### 7.1 Điểm mạnh

**Kiến trúc hiện đại**: Sử dụng YARP - thư viện reverse proxy được Microsoft phát triển và hỗ trợ chính thức, đảm bảo tính ổn định và cập nhật lâu dài.

**Cấu trúc routing rõ ràng**: Path pattern với `{**catch-all}` giúp đơn giản hóa việc forwarding request mà không cần viết custom transforms phức tạp.

**Tích hợp sẵn cross-cutting concerns**: Rate limiting, CORS, và health checks được tích hợp ngay từ đầu, giảm tải cho các microservices backend.

**Container-ready**: Đã có Dockerfile và docker-compose configuration, dễ dàng triển khai trong môi trường container.

### 7.2 Điểm cần cải thiện

**Authentication/Authorization**: Chưa có JWT validation hoặc authentication middleware. Tất cả requests đều được forward trực tiếp mà không qua authorization check.

**Single Destination per Cluster**: Mỗi service chỉ có một destination, không có redundancy nếu instance bị down.

**Rate Limiting**: Sử dụng in-memory rate limiting, không hoạt động trong distributed deployments với nhiều API Gateway instances.

**Circuit Breaker**: Chưa có circuit breaker pattern để handle cascading failures khi downstream services gặp vấn đề.

**Observability**: Chưa có structured logging, distributed tracing (OpenTelemetry), hoặc metrics collection.

### 7.3 Khuyến nghị cải thiện

**Ngắn hạn (1-2 tuần)**:
- Thêm JWT authentication middleware
- Cấu hình multiple destinations per cluster cho high availability
- Tích hợp circuit breaker (Yarp.ReverseProxy.CircuitBreaker)

**Trung hạn (1-2 tháng)**:
- Implement Redis-based distributed rate limiting
- Thêm OpenTelemetry cho distributed tracing
- Cấu hình structured logging với Serilog

**Dài hạn**:
- Implement service discovery (Consul, Eureka) thay vì hard-coded addresses
- Thêm API management features (quota, throttling per client)
- Implement GraphQL federation

---

## 8. Kết luận

API Gateway của ProgCoder Shop Microservices được xây dựng trên nền tảng vững chắc với YARP, cung cấp đầy đủ các tính năng cần thiết cho một hệ thống microservices. Kiến trúc hiện tại phù hợp cho giai đoạn development và có thể tiến tới production với một số cải thiện về resilience và observability.

Điểm mấu chốt để hệ thống hoạt động ổn định trong production là bổ sung authentication/authorization layer và cải thiện resilience với circuit breaker cùng multi-instance deployment.

---

## Phụ lục: API Gateway Endpoints

| Endpoint | Method | Mô tả |
|----------|--------|-------|
| `/` | GET | Health status info |
| `/health` | GET | Health check endpoint |
| `/basket-service/**` | * | Forward to basket-api |
| `/catalog-service/**` | * | Forward to catalog-api |
| `/discount-service/**` | * | Forward to discount-api |
| `/inventory-service/**` | * | Forward to inventory-api |
| `/notification-service/**` | * | Forward to notification-api |
| `/order-service/**` | * | Forward to order-api |
| `/report-service/**` | * | Forward to report-api |
| `/search-service/**` | * | Forward to search-api |
| `/communication-service/**` | * | Forward to communication-api |

---

**Tài liệu được tạo**: Phân tích dựa trên code từ `src/ApiGateway/YarpApiGateway/` và cấu hình Docker Compose.

**Phiên bản**: 1.0

**Ngày cập nhật**: Tháng 2/2026
