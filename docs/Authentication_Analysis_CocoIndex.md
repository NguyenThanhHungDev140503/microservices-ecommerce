# Báo cáo Phân tích Authentication System (sử dụng CocoIndex)

## Tổng quan
Hệ thống Authentication của dự án **Progcoder Shop Microservices** được xây dựng dựa trên **OAuth2 / OpenID Connect** với **Keycloak** làm Identity Provider.
Kiến trúc Microservices sử dụng JWT Bearer Token để xác thực và chuyển đổi claims thành `UserContext` để sử dụng trong tâng Application.

## 1. Cơ sở hạ tầng (Infrastructure)

### Config Keycloak (Docker Compose)
Dựa trên `docker-compose.infrastructure.yml`:
- **Image**: Keycloak (quay.io/keycloak/keycloak)
- **Port**: 8080
- **Database**: Postgres (kết nối chung với database chính)
- **Environment**:
    - `KC_HOSTNAME`: localhost
    - `KC_DB`: postgres
    - `WEBHOOK_URL`: (có vẻ dùng để trigger events sau khi user đăng ký/đăng nhập?)

### Service Extension (`AuthorizationExtension.cs`)
File: `src/Shared/BuildingBlocks/Authentication/Extensions/AuthorizationExtension.cs`

Extension method `AddAuthenticationAndAuthorization` thực hiện:
1. **AddAuthentication**: Default scheme là `JwtBearerDefaults.AuthenticationScheme`.
2. **AddJwtBearer**:
    - Load config từ `AuthorizationCfg` (Authority, Audience, ClientId).
    - `TokenValidationParameters`:
        - Validate Issuer, Lifetime, SigningKey.
        - `NameClaimType` = `sub`.
        - `RoleClaimType` = `role`.
    - **Custom Logic**: `OnTokenValidated` event handler:
        - Parse `realm_access` claim (đặc thù Keycloak).
        - Trích xuất `roles` từ JSON `realm_access`.
        - Add `ClaimTypes.Role` vào `ClaimsIdentity` để .NET nhận diện Roles chuẩn.

```csharp
// Snippet xử lý Keycloak Realm Roles
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
                id.AddClaim(new Claim(ClaimTypes.Role, role));
        }
    }
}
```

## 2. Mô hình người dùng (Context Model)

### UserContext (`UserContext.cs`)
File: `src/Shared/Common/Models/Context/UserContext.cs`

Class POCO chứa thông tin user sau khi parse từ token:
- `Id` (sub)
- `UserName`
- `Email`
- `FirstName`, `LastName`
- `Tenant`
- `Roles` (List<string>)

### Extension Method (`UserContextExtension.cs`)
File: `src/Shared/BuildingBlocks/Authentication/Extensions/UserContextExtension.cs`

Extension `GetCurrentUser` trên `IHttpContextAccessor`:
- Lấy `ClaimsPrincipal` từ `HttpContext.User`.
- Map các claims (`NameIdentifier`, `Email`, `GivenName`, `Role`, `Tenant`...) sang object `UserContext`.

```csharp
public static UserContext GetCurrentUser(this IHttpContextAccessor context)
{
    var identity = context.HttpContext?.User;
    // Map claims to properties...
    return new UserContext() { ... };
}
```

## 3. Sử dụng trong Application Layer

### Endpoint Integration
Ví dụ tìm thấy trong `CreateBrand.cs` (Catalog Service):
- Inject `IHttpContextAccessor`.
- Gọi `httpContext.GetCurrentUser()` để lấy thông tin người dùng hiện tại.
- Truyền thông tin user (Email) vào Domain Command/Actor để xử lý logic nghiệp vụ (Audit log, Permission check).

```csharp
// Snippet từ CreateBrand endpoint
private async Task<ApiCreatedResponse<Guid>> HandleCreateBrandAsync(
    ISender sender,
    IHttpContextAccessor httpContext,
    [FromBody] CreateBrandDto req)
{
    var currentUser = httpContext.GetCurrentUser();
    var command = new CreateBrandCommand(req, Actor.User(currentUser.Email));
    // ...
}
```

## Kết luận
Hệ thống sử dụng mô hình xác thực tập trung (Identity Server - Keycloak) và phân tán xác thực tại từng Microservice thông qua JWT.
- **Ưu điểm**:
    - Decoupled: Các services không cần gọi trực tiếp Identity Service để validate mỗi request (stateless verification).
    - Chuẩn hóa: Mapping roles từ Keycloak sang .NET Claims chuẩn giúp tận dụng `[Authorize(Roles="...")]` attribute của ASP.NET Core.
    - Context-aware: `UserContext` object giúp truy cập thông tin user dễ dàng trong code nghiệp vụ.

- **Điểm cần lưu ý**:
    - Token size: JWT chứa nhiều claims (đặc biệt là roles) có thể làm tăng header size.
    - Revocation: Stateless JWT khó thu hồi tức thì (dựa vào expiration time ngắn).
