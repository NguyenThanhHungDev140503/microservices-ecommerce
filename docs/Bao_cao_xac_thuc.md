# Báo Cáo Phân Tích Luồng Xác Thực

## A. Tóm Tắt Nhiệm Vụ

Nhiệm vụ này yêu cầu sử dụng công cụ MCP server code-graph-rag để thực hiện tìm kiếm toàn diện các chức năng xác thực trong codebase của dự án progcoder-shop-microservices, tập trung vào login, logout và cơ chế xác minh token. Mục tiêu là xác định chính xác tệp và dòng số nơi các chức năng này được định nghĩa, ánh xạ các mối quan hệ và phụ thuộc gọi giữa chúng, đồng thời cung cấp phân tích chi tiết về luồng xác thực bao gồm các đoạn mã liên quan.

Sau khi index toàn bộ codebase (1605 files, 5062 entities, 3732 relationships), tôi đã phân tích các thành phần xác thực chính và tạo báo cáo chi tiết về luồng xác thực dự án.

## B. Chi Tiết Triển Khai Mã Nguồn

### 1. Cấu Hình JWT Bearer Authentication

**File**: [`src/Shared/BuildingBlocks/Authentication/Extensions/AuthorizationExtension.cs`](src/Shared/BuildingBlocks/Authentication/Extensions/AuthorizationExtension.cs)  
**Dòng**: 21-86

**Mô tả**: File này chứa phương thức mở rộng `AddAuthenticationAndAuthorization` để cấu hình JWT Bearer authentication cho toàn bộ ứng dụng.

**Đoạn mã quan trọng**:

```csharp
public static IServiceCollection AddAuthenticationAndAuthorization(
    this IServiceCollection services,
    IConfiguration cfg)
{
    var authority = cfg[$"{AuthorizationCfg.Section}:{AuthorizationCfg.Authority}"];
    var clientId = cfg[$"{AuthorizationCfg.Section}:{AuthorizationCfg.ClientId}"];
    var audience = cfg[$"{AuthorizationCfg.Section}:{AuthorizationCfg.Audience}"];
    var requireHttps = cfg.GetValue<bool>($"{AuthorizationCfg.Section}:{AuthorizationCfg.RequireHttpsMetadata}");

    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options =>
    {
        options.Authority = authority;
        options.Audience = audience;
        options.RequireHttpsMetadata = requireHttps;
        options.SaveToken = true;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = false,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(60),
            NameClaimType = JwtRegisteredClaimNames.Sub,
            RoleClaimType = ClaimTypes.Role
        };

        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = ctx =>
            {
                var identity = ctx.Principal?.Identity as ClaimsIdentity;

                if (identity == null) return Task.CompletedTask;

                var realmAccess = ctx.Principal!.FindFirst(CustomClaimTypes.RealmAccess)?.Value;
                if (realmAccess != null)
                {
                    using var doc = JsonDocument.Parse(realmAccess);
                    if (doc.RootElement.TryGetProperty(CustomClaimTypes.Roles, out var roles))
                    {
                        foreach (var r in roles.EnumerateArray())
                        {
                            var role = r.GetString();
                            if (!string.IsNullOrEmpty(role))
                                identity.AddClaim(new Claim(ClaimTypes.Role, role));
                        }
                    }
                }

                return Task.CompletedTask;
            }
        };
    });

    services.AddAuthorization();

    return services;
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 25-28**: Đọc cấu hình từ appsettings.json bao gồm Authority (URL Keycloak), ClientId, Audience và RequireHttpsMetadata
- **Dòng 30-33**: Thiết lập JWT Bearer làm authentication scheme mặc định
- **Dòng 42-53**: Cấu hình các tham số xác minh token:
  - `ValidateIssuer = true`: Xác minh issuer của token
  - `ValidateAudience = false`: Không xác minh audience (cho phép nhiều audience)
  - `ValidateLifetime = true`: Xác minh thời hạn sống của token
  - `ValidateIssuerSigningKey = true`: Xác minh chữ ký của issuer
  - `ClockSkew = TimeSpan.FromSeconds(60)`: Cho phép sai lệch 60 giây giữa client và server
  - `NameClaimType = JwtRegisteredClaimNames.Sub`: Sử dụng claim "sub" làm tên người dùng
  - `RoleClaimType = ClaimTypes.Role`: Sử dụng claim "role" làm vai trò
- **Dòng 55-80**: Xử lý sự kiện `OnTokenValidated` để trích xuất các vai trò từ claim `realm_access` (JSON format) và thêm vào ClaimsIdentity. Điều này cho phép Keycloak trả về danh sách vai trò trong một claim thay vì phải thêm từng vai trò riêng lẻ.

**Quyết định thiết kế**: Sử dụng JWT Bearer authentication với Keycloak làm Identity Provider, cho phép xác minh token tự động và trích xuất thông tin vai trò từ claim realm_access.

### 2. Trích Xuất Thông Tin Người Dùng Từ Token

**File**: [`src/Shared/BuildingBlocks/Authentication/Extensions/UserContextExtension.cs`](src/Shared/BuildingBlocks/Authentication/Extensions/UserContextExtension.cs)  
**Dòng**: 16-39

**Mô tả**: File này chứa phương thức mở rộng `GetCurrentUser` để trích xuất thông tin người dùng từ HttpContext và trả về đối tượng UserContext.

**Đoạn mã quan trọng**:

```csharp
public static UserContext GetCurrentUser(this IHttpContextAccessor context)
{
    var identity = context.HttpContext?.User;
    var userId = identity?.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? string.Empty;
    var userName = identity?.FindFirst(CustomClaimTypes.UserName)?.Value ?? string.Empty;
    var firstName = identity?.FindFirst(ClaimTypes.GivenName)?.Value ?? string.Empty;
    var lastName = identity?.FindFirst(ClaimTypes.Surname)?.Value ?? string.Empty;
    var email = identity?.FindFirst(ClaimTypes.Email)?.Value ?? string.Empty;
    var tenant = identity?.FindFirst(CustomClaimTypes.Tenant)?.Value ?? string.Empty;
    var roles = identity?.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList() ?? [];
    bool.TryParse(identity?.FindFirst(CustomClaimTypes.EmailVerified)?.Value, out bool emailVerified);

    return new UserContext()
    {
        EmailVerified = emailVerified,
        FirstName = firstName,
        LastName = lastName,
        Email = email,
        Id = userId,
        UserName = userName,
        Tenant = tenant,
        Roles = roles
    };
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 18**: Nhận IHttpContextAccessor để truy cập HttpContext
- **Dòng 19**: Lấy ClaimsPrincipal từ HttpContext
- **Dòng 20-25**: Trích xuất các thông tin từ claims:
  - `NameIdentifier`: ID người dùng
  - `UserName`: Tên đăng nhập
  - `GivenName`: Tên đệm
  - `Surname`: Họ
  - `Email`: Email
  - `Tenant`: Tenant (cho multi-tenancy)
  - `Role`: Danh sách vai trò
  - `EmailVerified`: Email đã được xác minh hay chưa
- **Dòng 28-38**: Tạo và trả về đối tượng UserContext với tất cả thông tin đã trích xuất

**Quyết định thiết kế**: Sử dụng phương thức mở rộng (extension method) để cung cấp một cách tiện lợi để trích xuất thông tin người dùng từ token JWT. Điều này được sử dụng rộng rãi trong các endpoint của các microservices.

### 3. Interface Gọi API Keycloak

**File**: [`src/JobOrchestrator/App.Job/ApiClients/IKeycloakApi.cs`](src/JobOrchestrator/App.Job/ApiClients/IKeycloakApi.cs)  
**Dòng**: 10-26

**Mô tả**: Interface định nghĩa các phương thức để gọi API Keycloak, sử dụng thư viện Refit.

**Đoạn mã quan trọng**:

```csharp
public interface IKeycloakApi
{
    [Post("/realms/{realm}/protocol/openid-connect/token")]
    [Headers("Content-Type: application/x-www-form-urlencoded")]
    Task<KeycloakAccessToken> GetAccessTokenAsync(
        [AliasAs("realm")] string realm,
        [Body(BodySerializationMethod.UrlEncoded)] Dictionary<string, string> form);

    [Get("/admin/realms/{realm}/users/count")]
    Task<long> GetUsersCountAsync(
        [AliasAs("realm")] string realm,
        [Header("Authorization")] string bearerToken);
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 14-18**: Định nghĩa phương thức `GetAccessTokenAsync` để lấy access token từ Keycloak bằng OAuth2.0 client credentials grant
- **Dòng 20-24**: Định nghĩa phương thức `GetUsersCountAsync` để đếm số lượng người dùng trong realm

**Quyết định thiết kế**: Sử dụng Refit để tạo typed HTTP client, giúp giảm thiểu lỗi và tăng tính an toàn khi gọi API Keycloak.

### 4. Service Quản Lý Keycloak

**File**: [`src/Services/Notification/Core/Notification.Infrastructure/Services/KeycloakService.cs`](src/Services/Notification/Core/Notification.Infrastructure/Services/KeycloakService.cs)  
**Dòng**: 14-71

**Mô tả**: Service triển khai interface IKeycloakService để tương tác với Keycloak API.

**Đoạn mã quan trọng**:

```csharp
public sealed class KeycloakService(
    IKeycloakApi keycloakApi,
    IConfiguration cfg,
    IMapper mapper) : IKeycloakService
{
    public async Task<List<KeycloakUserDto>> GetUsersAsync(CancellationToken cancellationToken = default)
    {
        var accessToken = await GetAccessTokenAsync(cancellationToken);
        var realm = cfg[$"{KeycloakApiCfg.Section}:{KeycloakApiCfg.Realm}"]!;

        var result = await keycloakApi.GetUsersAsync(
            realm: realm,
            bearerToken: $"Bearer {accessToken.AccessToken}");

        return mapper.Map<List<KeycloakUserDto>>(result);
    }

    public async Task<List<KeycloakUserDto>> GetUsersByRoleAsync(string role, CancellationToken cancellationToken = default)
    {
        var accessToken = await GetAccessTokenAsync(cancellationToken);
        var realm = cfg[$"{KeycloakApiCfg.Section}:{KeycloakApiCfg.Realm}"]!;

        var result = await keycloakApi.GetUsersByRoleAsync(
            realm: realm,
            role: role,
            bearerToken: $"Bearer {accessToken.AccessToken}");

        return mapper.Map<List<KeycloakUserDto>>(result);
    }

    private async Task<KeycloakAccessToken> GetAccessTokenAsync(CancellationToken cancellationToken)
    {
        var realm = cfg[$"{KeycloakApiCfg.Section}:{KeycloakApiCfg.Realm}"]!;
        var clientId = cfg[$"{KeycloakApiCfg.Section}:{KeycloakApiCfg.ClientId}"]!;
        var clientSecret = cfg[$"{KeycloakApiCfg.Section}:{KeycloakApiCfg.ClientSecret}"]!;
        var grantType = cfg[$"{KeycloakApiCfg.Section}:{KeycloakApiCfg.GrantType}"]!;
        var scopes = cfg.GetRequiredSection($"{KeycloakApiCfg.Section}:{KeycloakApiCfg.Scopes}")
            .Get<string[]>() ?? throw new ArgumentNullException($"{KeycloakApiCfg.Section}:{KeycloakApiCfg.Scopes}");

        var form = new Dictionary<string, string>
        {
            { "client_id", clientId },
            { "client_secret", clientSecret },
            { "grant_type", grantType },
            { "scope", string.Join(" ", scopes) }
        };

        return await keycloakApi.GetAccessTokenAsync(realm, form);
    }
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 21-31**: Phương thức `GetUsersAsync` lấy access token và gọi API để lấy danh sách tất cả người dùng
- **Dòng 33-44**: Phương thức `GetUsersByRoleAsync` lấy access token và gọi API để lấy danh sách người dùng theo vai trò cụ thể
- **Dòng 50-68**: Phương thức riêng `GetAccessTokenAsync` để lấy access token từ Keycloak bằng client credentials:
  - Đọc cấu hình từ appsettings.json
  - Tạo form data với client_id, client_secret, grant_type và scope
  - Gọi API Keycloak để lấy access token

**Quyết định thiết kế**: Sử dụng service pattern để đóng gói logic gọi API Keycloak, giúp dễ kiểm thử và bảo trì. Service tự động quản lý việc lấy và sử dụng access token.

### 5. Service Gọi API Catalog với Token Keycloak

**File**: [`src/Services/Inventory/Core/Inventory.Infrastructure/Services/CatalogApiService.cs`](src/Services/Inventory/Core/Inventory.Infrastructure/Services/CatalogApiService.cs)  
**Dòng**: 79-90

**Mô tả**: Service triển khai ICatalogApiService để gọi API Catalog service, sử dụng token từ Keycloak.

**Đoạn mã quan trọng**:

```csharp
public async Task<ApiGetResponse<GetProductByIdResponse>?> GetProductByIdAsync(string productId)
{
    try
    {
        var tokenResponse = await GetAccessTokenAsync();
        var response = await _catalogApi.GetProductByIdAsync(productId, $"Bearer {tokenResponse.AccessToken}");
        return response;
    }
    catch (ApiException ex)
    {
        _logger.LogWarning(ex, "Error calling Catalog API for productId: {ProductId}", productId);
        return null;
    }
}

private async Task<KeycloakAccessTokenResponse> GetAccessTokenAsync()
{
    var form = new Dictionary<string, string>
    {
        { "client_id", _clientId },
        { "client_secret", _clientSecret },
        { "grant_type", _grantType },
        { "scope", string.Join(" ", _scopes!) }
    };

    return await _keycloakApi.GetAccessTokenAsync(_realm, form);
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 64-65**: Lấy access token từ Keycloak trước khi gọi API
- **Dòng 65**: Sử dụng access token để gọi API Catalog service với Bearer authentication
- **Dòng 68-72**: Xử lý exception và log warning nếu có lỗi

**Quyết định thiết kế**: Service tự động quản lý việc lấy và sử dụng access token khi gọi API bên ngoài, giúp giảm thiểu code và tăng tính an toàn.

### 6. Model Keycloak Access Token

**File**: [`src/JobOrchestrator/App.Job/Models/Keycloaks/KeycloakAccessToken.cs`](src/JobOrchestrator/App.Job/Models/Keycloaks/KeycloakAccessToken.cs)  
**Dòng**: 9-35

**Mô tả**: Model đại diện cho access token trả về từ Keycloak.

**Đoạn mã quan trọng**:

```csharp
public sealed class KeycloakAccessToken
{
    [JsonPropertyName("access_token")]
    public string? AccessToken { get; set; }

    [JsonPropertyName("expires_in")]
    public int ExpiresIn { get; set; }

    [JsonPropertyName("refresh_expires_in")]
    public int RefreshExpiresIn { get; set; }

    [JsonPropertyName("token_type")]
    public string? TokenType { get; set; }

    [JsonPropertyName("id_token")]
    public string? IdToken { get; set; }

    [JsonPropertyName("not-before-policy")]
    public int NotBeforePolicy { get; set; }

    [JsonPropertyName("scope")]
    public string? Scope { get; set; }
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 13-14**: Access token để xác minh request
- **Dòng 16-17**: Thời gian hết hạn của access token (tính bằng giây)
- **Dòng 19-20**: Thời gian hết hạn của refresh token
- **Dòng 22-23**: Loại token (thường là "Bearer")
- **Dòng 25-26**: ID token để lấy thông tin người dùng
- **Dòng 28-29**: Chính sách không sử dụng token trước thời gian này
- **Dòng 31-32**: Scope của token

**Quyết định thiết kế**: Sử dụng System.Text.Json.Serialization để deserialize JSON response từ Keycloak API, giúp dễ dàng chuyển đổi giữa JSON và object C#.

### 7. Cấu Hình Authentication

**File**: [`src/Shared/Common/Configurations/AuthorizationCfg.cs`](src/Shared/Common/Configurations/AuthorizationCfg.cs)  
**Dòng**: 1-25

**Mô tả**: Class chứa các hằng số cấu hình cho authentication.

**Đoạn mã quan trọng**:

```csharp
public sealed class AuthorizationCfg
{
    public const string Section = "Authentication";

    public const string Authority = "Authority";
    public const string Audience = "Audience";
    public const string ClientId = "ClientId";
    public const string ClientSecret = "ClientSecret";
    public const string Scopes = "Scopes";
    public const string RequireHttpsMetadata = "RequireHttpsMetadata";
    public const string OAuth2RedirectUrl = "OAuth2RedirectUrl";
}
```

**Giải thích logic và quyết định thiết kế**:

- **Dòng 7**: Tên section trong appsettings.json
- **Dòng 9**: URL của Keycloak (issuer)
- **Dòng 11**: Audience của token
- **Dòng 13**: Client ID để lấy access token
- **Dòng 15**: Client secret để lấy access token
- **Dòng 17**: Scopes của token
- **Dòng 19**: Yêu cầu HTTPS metadata hay không
- **Dòng 21**: URL redirect cho OAuth2

**Quyết định thiết kế**: Sử dụng const string để định nghĩa các key cấu hình, giúp tránh lỗi đánh máy và tăng tính an toàn.

## C. Kiểm Thử

### 1. Unit Tests

Không tìm thấy unit tests cụ thể cho các chức năng xác thực trong codebase. Tuy nhiên, có thể kiểm thử bằng cách:

- **Kiểm thử cấu hình JWT**: Tạo unit test để xác minh rằng các tham số TokenValidationParameters được cấu hình đúng
- **Kiểm thử trích xuất UserContext**: Tạo unit test để xác minh rằng phương thức GetCurrentUser trích xuất đúng thông tin từ claims
- **Kiểm thử lấy Access Token**: Tạo unit test để xác minh rằng phương thức GetAccessTokenAsync gọi đúng API Keycloak và trả về token hợp lệ

### 2. Integration Tests

- **Kiểm thử luồng xác thực hoàn chỉnh**: Tạo integration test để kiểm thử toàn bộ luồng từ login đến trích xuất thông tin người dùng
- **Kiểm thử gọi API với token**: Tạo integration test để kiểm thử việc gọi API bên ngoài với access token từ Keycloak

### 3. Manual Tests

- **Kiểm thử đăng nhập và lấy token**: Sử dụng Postman hoặc curl để đăng nhập vào Keycloak và lấy access token
- **Kiểm thử xác minh token**: Sử dụng access token để gọi API của các microservices và xác minh rằng token được chấp nhận
- **Kiểm thử trích xuất thông tin người dùng**: Gọi API endpoint và kiểm thử rằng thông tin người dùng được trích xuất đúng

## D. Thách Thức và Giải Pháp

### 1. Không Tìm Thấy Các Hàm Login/Logout Cụ Thể

**Vấn đề**: Codebase không chứa các hàm login/logout cụ thể vì sử dụng Keycloak làm Identity Provider. Người dùng đăng nhập/logout trực tiếp qua Keycloak UI.

**Giải pháp**: Sử dụng Keycloak UI để đăng nhập/logout. Các microservices chỉ cần xác minh token từ Keycloak.

### 2. Không Tìm Thấy Hàm Refresh Token

**Vấn đề**: Không tìm thấy hàm refresh token để làm mới access token khi hết hạn.

**Giải pháp**: Có thể cần thêm hàm refresh token để làm mới access token một cách minh bạch, hoặc sử dụng refresh token từ Keycloak.

### 3. Không Tìm Thấy Hàm Logout

**Vấn đề**: Không tìm thấy hàm logout để hủy session người dùng.

**Giải pháp**: Sử dụng Keycloak UI để logout, hoặc thêm hàm logout để hủy token trên client side.

### 4. Không Tìm Thấy Unit Tests

**Vấn đề**: Không tìm thấy unit tests cho các chức năng xác thực.

**Giải pháp**: Cần thêm unit tests để đảm bảo chất lượng và độ tin cậy của mã nguồn xác thực.

## E. Cải Tiến và Tối Ưu Hóa

### 1. Sử Dụng JWT Bearer Authentication

**Cải tiến**: Sử dụng JWT Bearer authentication thay vì nó là tiêu chuẩn, an toàn và được hỗ trợ rộng rãi trong ASP.NET Core.

**Lợi ích**: Giảm thiểu code, tăng tính an toàn, dễ dàng tích hợp với các Identity Provider khác.

### 2. Sử Dụng Refit để Gọi API

**Cải tiến**: Sử dụng thư viện Refit để tạo typed HTTP client.

**Lợi ích**: Giảm thiểu lỗi, tăng tính an toàn, dễ dàng kiểm thử.

### 3. Sử Dụng Extension Method

**Cải tiến**: Sử dụng extension method để cung cấp các chức năng tiện lợi như GetCurrentUser.

**Lợi ích**: Giảm thiểu code, tăng tính tái sử dụng, dễ dàng bảo trì.

### 4. Sử Dụng OnTokenValidated Event

**Cải tiến**: Sử dụng sự kiện OnTokenValidated để trích xuất các vai trò từ claim realm_access.

**Lợi ích**: Cho phép Keycloak trả về danh sách vai trò trong một claim thay vì phải thêm từng vai trò riêng lẻ, giảm kích thước token.

## F. Công Cụ và Công Nghệ Sử Dụng

### 1. Phát Triển

- **Ngôn ngữ**: C# (.NET 6/7/8)
- **Framework**: ASP.NET Core
- **Authentication**: JWT Bearer Authentication
- **Identity Provider**: Keycloak
- **HTTP Client**: Refit
- **Serialization**: System.Text.Json

### 2. Kiểm Thử

- **Testing Framework**: xUnit (có thể được sử dụng)
- **Integration Testing**: Có thể sử dụng TestServer hoặc WebApplicationFactory

### 3. Triển Khai

- **Containerization**: Docker
- **Orchestration**: Docker Compose
- **Reverse Proxy**: YARP (Yet Another Reverse Proxy) trong API Gateway

### 4. Giám Sát và Ghi Nhập Ký

- **Logging**: Serilog
- **Tracing**: OpenTelemetry
- **Monitoring**: Prometheus, Grafana

### 5. Khác

- **API Gateway**: YARP API Gateway
- **Message Broker**: RabbitMQ
- **Cache**: Redis
- **Database**: PostgreSQL

## Kết Luận

Luồng xác thực của dự án progcoder-shop-microservices sử dụng Keycloak làm Identity Provider với JWT Bearer authentication. Các thành phần chính bao gồm:

1. **Cấu hình JWT Bearer Authentication**: [`AuthorizationExtension.cs`](src/Shared/BuildingBlocks/Authentication/Extensions/AuthorizationExtension.cs) cấu hình JWT Bearer authentication với các tham số xác minh token và xử lý sự kiện OnTokenValidated để trích xuất vai trò.

2. **Trích Xuất Thông Tin Người Dùng**: [`UserContextExtension.cs`](src/Shared/BuildingBlocks/Authentication/Extensions/UserContextExtension.cs) cung cấp phương thức GetCurrentUser để trích xuất thông tin người dùng từ token.

3. **Lấy Access Token từ Keycloak**: Các service như [`KeycloakService.cs`](src/Services/Notification/Core/Notification.Infrastructure/Services/KeycloakService.cs) và [`CatalogApiService.cs`](src/Services/Inventory/Core/Inventory.Infrastructure/Services/CatalogApiService.cs) sử dụng [`IKeycloakApi.cs`](src/JobOrchestrator/App.Job/ApiClients/IKeycloakApi.cs) để lấy access token từ Keycloak bằng OAuth2.0 client credentials grant.

4. **Gọi API với Token**: Các service sử dụng access token để gọi API bên ngoài với Bearer authentication.

5. **Cấu Hình**: Các cấu hình được định nghĩa trong [`AuthorizationCfg.cs`](src/Shared/Common/Configurations/AuthorizationCfg.cs) và [`KeycloakApiCfg.cs`](src/Shared/Common/Configurations/KeycloakApiCfg.cs) (không được đọc nhưng được tham chiếu trong code).

Luồng xác thực này là an toàn, có thể mở rộng và dễ dàng bảo trì. Tuy nhiên, cần thêm unit tests và integration tests để đảm bảo chất lượng và độ tin cậy.
