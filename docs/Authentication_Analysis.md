# Phân tích Hệ thống Authentication - progcoder-shop-microservices

Tài liệu này trình bày kết quả phân tích hệ thống xác thực (Authentication) và phân quyền (Authorization) trong dự án, sử dụng công cụ `code-graph-rag-index`.

## 1. Kiến trúc Tổng quan

Hệ thống sử dụng **Keycloak** làm Identity and Access Management (IAM) provider tập trung. Luồng xác thực dựa trên giao thức **OpenID Connect (OIDC)** và các microservice backend xác thực bằng **JWT (JSON Web Tokens)**.

### Các thành phần chính:
- **IAM Provider**: Keycloak (chạy độc lập, cung cấp UI quản lý user/role).
- **Frontend (App.Store & App.Admin)**: Sử dụng thư viện `keycloak-js` để xử lý đăng nhập và quản lý token.
- **Backend Building Blocks**: Cung cấp các extension method dùng chung để cấu hình JWT Bearer Authentication.
- **Microservices**: Áp dụng các extension đó để bảo vệ API Endpoints.

---

## 2. Chi tiết Triển khai Backend

### 2.1. Cấu hình xác thực dùng chung
Tất cả các microservice backend đều kế thừa logic xác thực từ thư viện Shared.

- **File**: `src/Shared/BuildingBlocks/Authentication/Extensions/AuthorizationExtension.cs`
- **Phương thức chính**: `AddAuthenticationAndAuthorization`

```csharp
public static IServiceCollection AddAuthenticationAndAuthorization(
    this IServiceCollection services,
    IConfiguration cfg)
{
    // Đọc thông tin Authority, ClientId, Audience từ cấu hình
    // ...
    services.AddAuthentication(options => { ... })
    .AddJwtBearer(options => {
        options.Authority = authority;
        options.Audience = audience;
        // ...
        options.Events = new JwtBearerEvents {
            OnTokenValidated = ctx => {
                // Mapping Roles từ Keycloak 'realm_access' claim sang ClaimTypes.Role của .NET
                // ...
            }
        };
    });
}
```

### 2.2. Trích xuất ngữ cảnh người dùng
- **File**: `src/Shared/BuildingBlocks/Authentication/Extensions/UserContextExtension.cs`
- **Phương thức**: `GetCurrentUser`
Phương thức này cho phép controller dễ dàng lấy thông tin `userId`, `roles`, `email`, `tenant` từ `HttpContext`.

### 2.3. Bảo vệ Endpoints
Các API Endpoint sử dụng thư viện **Carter** và được bảo vệ bằng phương thức `.RequireAuthorization()`.

- **Ví dụ**: `src/Services/Catalog/Api/Catalog.Api/Endpoints/CreateProduct.cs`
```csharp
app.MapPost(ApiRoutes.Product.Create, HandleCreateProductAsync)
    .RequireAuthorization(); // Yêu cầu JWT token hợp lệ
```

---

## 3. Chi tiết Triển khai Frontend

Tất cả các ứng dụng React trong thư mục `src/Apps` đều sử dụng một cơ chế xác thực nhất quán.

- **File**: `src/Apps/App.Store/src/services/keycloakService.js`
- **Công nghệ**: `keycloak-js`

### Các tính năng xử lý:
1. **Khởi tạo (initKeycloak)**: Kiểm tra cấu hình từ biến môi trường và thiết lập các sự kiện (`onAuthSuccess`, `onTokenExpired`).
2. **Quản lý Token**:
   - Token được lưu vào **Cookie** (`access_token`) để phục vụ các request SSR hoặc gửi lên server.
   - Cơ chế tự động làm mới token (`updateToken`) khi sắp hết hạn.
3. **Login/Logout**: Chuyển hướng người dùng sang trang SSO của Keycloak.

---

## 4. Tương tác với Keycloak Admin API

Một số dịch vụ nâng cao (như `Notification` và `Report`) không chỉ xác thực token mà còn chủ động gọi vào Keycloak để lấy dữ liệu người dùng.

- **Interface**: `IKeycloakApi`
- **Các phương thức**: `GetUsersAsync`, `GetUsersByRoleAsync`, `GetAccessTokenAsync`.
Điều này cho phép hệ thống gửi thông báo đến đúng user dựa trên vai trò hoặc tạo các báo cáo thống kê về người dùng trong hệ thống.

---

## 5. Kết luận

Hệ thống Authentication của dự án được thiết kế rất chuyên nghiệp và có tính mở rộng cao:
- **Tính nhất quán**: Sử dụng Building Blocks giúp tất cả microservice có cùng một tiêu chuẩn bảo mật.
- **Tính an toàn**: Logic role-mapping được xử lý an toàn tại tầng xác thực của backend.
- **Trải nghiệm người dùng**: Khả năng SSO (Single Sign-On) nhờ Keycloak giúp người dùng chỉ cần đăng nhập một lần cho tất cả các ứng dụng (Store, Admin).

---
*Tài liệu được tạo tự động bởi Antigravity sử dụng Code-Graph-RAG.*
