# Kiến trúc Giao tiếp Microservices - ProGCoder Shop

## Tổng quan

Dự án ProGCoder Shop là một hệ thống thương mại điện tử được xây dựng theo kiến trúc microservices hiện đại, sử dụng nhiều phương thức giao tiếp khác nhau để đảm bảo tính linh hoạt, khả năng mở rộng và hiệu suất cao.

## 1. Kiến trúc Hệ thống

Hệ thống bao gồm các thành phần chính sau:

- **Các dịch vụ backend** (trong thư mục `src/Services`)
- **API Gateway** (YARP) làm điểm truy cập duy nhất
- **Các ứng dụng frontend** (React) tiêu thụ API thông qua gateway
- **Cơ sở hạ tầng hỗ trợ** (RabbitMQ, Redis, PostgreSQL, v.v.)

## 2. Các Dịch vụ Backend

Dưới đây là danh sách các dịch vụ backend chính trong hệ thống:

### 2.1. Dịch vụ Chính

1. **Catalog Service** (`src/Services/Catalog`)
   - Quản lý thông tin sản phẩm
   - Cung cấp API để tìm kiếm và lọc sản phẩm

2. **Basket Service** (`src/Services/Basket`)
   - Quản lý giỏ hàng người dùng
   - Xử lý quy trình thanh toán giỏ hàng

3. **Order Service** (`src/Services/Order`)
   - Quản lý đơn hàng
   - Xử lý quy trình tạo và cập nhật trạng thái đơn hàng

4. **Inventory Service** (`src/Services/Inventory`)
   - Quản lý hàng tồn kho
   - Xử lý việc trừ/xuất hàng khi có đơn hàng

5. **Discount Service** (`src/Services/Discount`)
   - Quản lý mã giảm giá và chương trình khuyến mãi
   - Áp dụng quy tắc giảm giá

### 2.2. Dịch vụ Hỗ trợ

6. **Notification Service** (`src/Services/Notification`)
   - Gửi thông báo qua email, SMS, push notification

7. **Search Service** (`src/Services/Search`)
   - Cung cấp chức năng tìm kiếm nâng cao
   - Đồng bộ dữ liệu tìm kiếm

8. **Report Service** (`src/Services/Report`)
   - Tạo báo cáo kinh doanh
   - Thống kê và phân tích dữ liệu

9. **Communication Service** (`src/Services/Communication`)
   - Quản lý liên lạc với khách hàng
   - Xử lý các kênh truyền thông khác nhau

10. **Identity Service** (`src/Services/Identity`)
    - Xử lý xác thực và ủy quyền
    - Quản lý người dùng và vai trò

## 3. Mô hình Giao tiếp Dịch vụ

### 3.1. Giao tiếp gRPC

Hệ thống sử dụng gRPC cho các cuộc gọi đồng bộ hiệu suất cao giữa các dịch vụ. Các file proto định nghĩa các giao diện dịch vụ:

- `src/Shared/Contracts/Catalog.Contract/Protos/catalog.proto`
- `src/Shared/Contracts/Discount.Contract/Protos/discount.proto`
- `src/Shared/Contracts/Inventory.Contract/Protos/inventory.proto`
- `src/Shared/Contracts/Order.Contract/Protos/order.proto`
- `src/Shared/Contracts/Report.Contract/Protos/report.proto`

#### Mối quan hệ gRPC giữa các dịch vụ:

- **Order Service** sử dụng các gRPC client để:
  - Gọi Catalog Service để lấy thông tin sản phẩm
  - Gọi Discount Service để xác thực mã giảm giá
  - Gọi Payment Service để xử lý thanh toán
  - Gọi Basket Service để lấy chi tiết giỏ hàng

- **Inventory Service** sử dụng Catalog gRPC client để lấy thông tin sản phẩm

- **Notification Service** sử dụng Order gRPC client để lấy thông tin đơn hàng

- **Report Service** sử dụng Order gRPC client để lấy dữ liệu báo cáo

- **Communication Service** sử dụng Order gRPC client để lấy thông tin đơn hàng

- **Search Service** sử dụng Catalog gRPC client để đồng bộ dữ liệu sản phẩm

### 3.2. Giao tiếp Asynchronous với MassTransit

Hệ thống sử dụng MassTransit với RabbitMQ cho giao tiếp bất đồng bộ dựa trên sự kiện:

#### Các sự kiện chính:

1. **BasketCheckoutIntegrationEvent**
   - Được phát ra bởi Basket Service khi người dùng tiến hành thanh toán
   - Được tiêu thụ bởi Order Service để bắt đầu quy trình tạo đơn hàng

2. **OrderCreatedIntegrationEvent**
   - Được phát ra bởi Order Service sau khi đơn hàng được tạo thành công
   - Được tiêu thụ bởi:
     - Inventory Service để trừ kho
     - Discount Service để áp dụng quy tắc giảm giá
     - Notification Service để gửi thông báo
     - Search Service để cập nhật chỉ mục tìm kiếm
     - Report Service để tạo báo cáo
     - Communication Service để xử lý truyền thông

3. **OrderStatusChangedIntegrationEvent**
   - Được phát ra khi trạng thái đơn hàng thay đổi
   - Được tiêu thụ bởi Notification Service để gửi thông báo trạng thái

4. **InsufficientStockIntegrationEvent**
   - Được phát ra bởi Inventory Service khi không đủ hàng tồn kho
   - Được tiêu thụ bởi Order Service để xử lý trường hợp hết hàng

5. **OrderCancelledIntegrationEvent**
   - Được phát ra khi đơn hàng bị hủy
   - Được tiêu thụ bởi Inventory Service để hoàn trả hàng tồn kho

6. **OrderDeliveredIntegrationEvent**
   - Được phát ra khi đơn hàng được giao thành công
   - Được tiêu thụ bởi Inventory Service để cập nhật trạng thái tồn kho

7. **SendEmailIntegrationEvent**
   - Được phát ra bởi Communication Service
   - Được tiêu thụ bởi Notification Service để xử lý gửi email

## 4. Cấu trúc Giao tiếp Frontend-Backend

### 4.1. API Gateway (YARP)

Tệp cấu hình: `src/ApiGateway/YarpApiGateway/appsettings.json`

Gateway định tuyến các yêu cầu từ frontend đến các dịch vụ backend tương ứng:

- `/catalog-service/*` → Catalog API service
- `/basket-service/*` → Basket API service
- `/order-service/*` → Order API service
- `/discount-service/*` → Discount API service
- `/notification-service/*` → Notification API service
- `/search-service/*` → Search API service
- `/report-service/*` → Report API service
- `/communication-service/*` → Communication API service
- `/information-service/*` → Information API service
- `/payment-service/*` → Payment API service

### 4.2. Ứng dụng Frontend

#### App.Store (`src/Apps/App.Store`)

- Sử dụng URL Gateway: `http://localhost:15009` (theo `.env.sample`)
- Sử dụng axios instance với interceptor để thêm JWT token từ cookie vào header
- Kết nối tới Keycloak để xác thực người dùng

#### App.Admin (`src/Apps/App.Admin`)

- Sử dụng URL Gateway: `http://localhost:15009` (theo `.env.sample`)
- Cơ chế xác thực tương tự App.Store
- Cung cấp giao diện quản trị cho hệ thống

### 4.3. Cơ chế Xác thực

- Cả hai ứng dụng frontend đều sử dụng cookie `access_token` để xác thực với gateway
- Interceptor được định nghĩa trong tệp `axiosInstance.js` để thêm token vào mỗi yêu cầu
- Keycloak được sử dụng làm hệ thống xác thực tập trung cho toàn hệ thống

## 5. Cấu hình Chung trong Shared Kernel

### 5.1. Cấu hình gRPC

Tệp: `src/Shared/Common/Configurations/GrpcClientCfg.cs`

Cung cấp cấu hình chung cho các gRPC client bao gồm:
- Thiết lập kênh gRPC
- Cấu hình các tùy chọn kết nối
- Cấu hình các interceptor như ApiKeyValidationInterceptor

### 5.2. Cấu hình Message Broker

Tệp: `src/Shared/Common/Configurations/MessageBrokerCfg.cs`

Cung cấp cấu hình chung cho MassTransit với RabbitMQ bao gồm:
- Cấu hình kết nối RabbitMQ
- Cấu hình serializer
- Cấu hình consumers và publishers

## 6. Mô hình Tích hợp Sự kiện

Dưới đây là chuỗi sự kiện xảy ra khi một đơn hàng được tạo:

1. Người dùng nhấn nút "Checkout" trên App.Store
2. App.Store gửi yêu cầu đến `/basket-service/checkout` thông qua API Gateway
3. Basket Service publish sự kiện `BasketCheckoutIntegrationEvent`
4. Order Service consume sự kiện và bắt đầu quá trình tạo đơn hàng
5. Order Service gọi các gRPC client để:
   - Gọi Basket gRPC client để lấy chi tiết giỏ hàng
   - Gọi Catalog gRPC client để xác minh sản phẩm
   - Gọi Discount gRPC client để xác thực mã giảm giá
   - Gọi Payment gRPC client để xử lý thanh toán
6. Order Service lưu trữ đơn hàng và publish `OrderCreatedIntegrationEvent`
7. Các dịch vụ khác consume sự kiện:
   - Inventory Service trừ kho và cập nhật tồn kho
   - Discount Service áp dụng quy tắc giảm giá
   - Notification Service gửi thông báo cho khách hàng
   - Search Service cập nhật chỉ mục tìm kiếm
   - Report Service tạo báo cáo bán hàng
   - Communication Service xử lý truyền thông với khách hàng

## 7. Tóm tắt Mô hình Giao tiếp

| Dịch vụ | Giao tiếp gRPC (Client) | Giao tiếp MassTransit (Consumer) | Giao tiếp MassTransit (Publisher) |
|--------|--------------------------|----------------------------------|----------------------------------|
| Basket | - | `BasketCheckoutIntegrationEvent` | `BasketCheckoutIntegrationEvent` |
| Order | Catalog, Discount, Payment, Basket | `BasketCheckoutIntegrationEvent`, `InsufficientStockIntegrationEvent` | `OrderCreatedIntegrationEvent`, `OrderStatusChangedIntegrationEvent` |
| Inventory | Catalog | `OrderCreatedIntegrationEvent`, `OrderCancelledIntegrationEvent`, `OrderDeliveredIntegrationEvent` | `InsufficientStockIntegrationEvent` |
| Discount | - | `OrderCreatedIntegrationEvent` | - |
| Notification | Order | `OrderCreatedIntegrationEvent`, `OrderStatusChangedIntegrationEvent`, `SendEmailIntegrationEvent` | - |
| Search | Catalog | `OrderCreatedIntegrationEvent` | `ProductSearchIntegrationEvent` |
| Report | Order | `OrderCreatedIntegrationEvent` | - |
| Communication | Order, Information | `OrderCreatedIntegrationEvent` | `SendEmailIntegrationEvent` |

## 8. Kết luận

Kiến trúc giao tiếp của ProGCoder Shop thể hiện một hệ thống microservices hiện đại với các đặc điểm nổi bật:

- **Tách biệt rõ ràng**: Mỗi dịch vụ có trách nhiệm riêng biệt và giao tiếp thông qua các giao diện được định nghĩa rõ ràng
- **Tính mở rộng cao**: Sử dụng cả gRPC đồng bộ và MassTransit bất đồng bộ cho phép mở rộng độc lập từng dịch vụ
- **Tính khả dụng cao**: Sử dụng API Gateway để cung cấp một điểm truy cập duy nhất cho frontend, giúp quản lý dễ dàng hơn
- **Xử lý sự kiện mạnh mẽ**: Kiến trúc dựa trên sự kiện cho phép hệ thống phản hồi nhanh chóng và hiệu quả với các thay đổi
- **Xác thực tập trung**: Sử dụng Keycloak cho xác thực và ủy quyền toàn hệ thống

Mô hình này cho phép mỗi dịch vụ phát triển, triển khai và mở rộng độc lập trong khi vẫn duy trì được tính toàn vẹn và nhất quán của hệ thống tổng thể.