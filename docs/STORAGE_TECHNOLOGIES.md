# Các Công Nghệ Lưu Trữ Trong Dự Án ProgCoder Shop Microservices

## Tổng Quan

Dự án ProgCoder Shop Microservices sử dụng **polyglot persistence** - kiến trúc đa dạng hóa công nghệ lưu trữ, trong đó mỗi service chọn công nghệ phù hợp nhất với use case của mình.

```mermaid
flowchart TB
    subgraph "Storage Technologies Overview"
        direction TB
        
        subgraph "Relational Databases"
            PG[(PostgreSQL)]
            SQL[(SQL Server)]
            MY[(MySQL)]
        end
        
        subgraph "NoSQL Databases"
            MG[(MongoDB)]
            ES[(Elasticsearch)]
        end
        
        subgraph "Cache & Message Broker"
            RD[(Redis)]
            RMQ[(RabbitMQ)]
        end
        
        subgraph "Object Storage"
            MIN[(MinIO)]
        end
        
        subgraph "Observability"
            LO[(Loki)]
            TE[(Tempo)]
            PR[(Prometheus)]
            GF[(Grafana)]
        end
    end
```

---

## 1. Relational Databases

### 1.1. PostgreSQL

**Thông tin cơ bản:**
- **Image:** postgres:latest
- **Port:** 5432
- **Container:** postgres-sql
- **User:** postgres
- **Password:** 123456789Aa

**Các Service sử dụng:**
| Service | Database Name | Mục đích |
|---------|--------------|----------|
| Catalog Service | Catalog_Service | Lưu trữ products, categories, brands |
| Keycloak | keycloak_db | Lưu trữ user identities, roles |

**Connection String:**
```json
{
  "ConnectionStrings": {
    "DbType": "POSTGRESQL",
    "Database": "Server=postgres-sql;Port=5432;Database=Catalog_Service;User Id=postgres;Password=123456789Aa;Include Error Detail=true;"
  }
}
```

**Lý do chọn PostgreSQL:**
- ✅ **ACID Compliance** - Đảm bảo tính toàn vẹn dữ liệu
- ✅ **JSON Support** - Hỗ trợ lưu trữ và truy vấn JSON
- ✅ **Full-text Search** - Tích hợp tìm kiếm văn bản
- ✅ **Marten Integration** - Event sourcing với Marten library

---

### 1.2. SQL Server

**Thông tin cơ bản:**
- **Image:** mcr.microsoft.com/mssql/server:latest
- **Port:** 1433
- **Container:** sql-server
- **User:** sa
- **Password:** 123456789Aa

**Các Service sử dụng:**
| Service | Database Name | Mục đích |
|---------|--------------|----------|
| Order Service | Order_Service | Lưu trữ orders, order items |

**Connection String:**
```json
{
  "ConnectionStrings": {
    "DbType": "SQLSERVER",
    "Database": "Server=sql-server,1433;Database=Order_Service;User Id=sa;Password=123456789Aa;Encrypt=False;TrustServerCertificate=True;"
  }
}
```

**Lý do chọn SQL Server:**
- ✅ **Enterprise Features** - Row-level security, encryption
- ✅ **EF Core Support** - Integration tốt với Entity Framework
- ✅ **Transaction Support** - Distributed transactions

---

### 1.3. MySQL

**Thông tin cơ bản:**
- **Image:** mysql:latest
- **Port:** 3306
- **Container:** mysql
- **Root Password:** 123456789Aa

**Các Service sử dụng:**
| Service | Database Name | Mục đích |
|---------|--------------|----------|
| Inventory Service | Inventory_Service | Lưu trữ inventory items, reservations |

**Lý do chọn MySQL:**
- ✅ **High Performance** - Tốc độ đọc/ghi cao
- ✅ **Widely Used** - Cộng đồng lớn, tài liệu phong phú
- ✅ **Replication** - Dễ dàng setup master-slave

---

## 2. NoSQL Databases

### 2.1. MongoDB

**Thông tin cơ bản:**
- **Image:** mongo:latest
- **Port:** 27017
- **Container:** mongodb
- **Root Username:** admin
- **Root Password:** 123456789Aa

**Các Service sử dụng:**
| Service | Database Name | Mục đích |
|---------|--------------|----------|
| Basket Service | Basket_Service | Shopping cart data |
| Report Service | Report_Service | Dashboard data, analytics |
| Discount Service | Discount_Service | Coupons, promotions |
| Notification Service | Notification_Service | Notification logs |

**Connection String:**
```json
{
  "ConnectionStrings": {
    "DbType": "MONGODB",
    "DatabaseName": "Basket_Service",
    "Database": "mongodb://mongodb:123456789Aa@mongodb:27017"
  }
}
```

**Lý do chọn MongoDB:**
- ✅ **Flexible Schema** - Phù hợp với dữ liệu thay đổi thường xuyên
- ✅ **Horizontal Scaling** - Sharding cho dữ liệu lớn
- ✅ **JSON Native** - Lưu trữ document dạng JSON/BSON
- ✅ **High Write Throughput** - Ghi dữ liệu nhanh

---

### 2.2. Elasticsearch

**Thông tin cơ bản:**
- **Image:** docker.elastic.co/elasticsearch/elasticsearch:8.x
- **Port:** 9200
- **Container:** elasticsearch
- **Security:** xpack.security.enabled = true

**Các Service sử dụng:**
| Service | Index Name | Mục đích |
|---------|-----------|----------|
| Search Service | products | Full-text search |

**Configuration:**
```yaml
environment:
  ELASTIC_PASSWORD: ${ELASTIC_PASSWORD}
  xpack.security.enabled: true
  discovery.type: single-node
  bootstrap.memory_lock: "true"
```

**Lý do chọn Elasticsearch:**
- ✅ **Full-text Search** - Tìm kiếm văn bản mạnh mẽ
- ✅ **Fuzzy Matching** - Tìm kiếm gần đúng, typo tolerance
- ✅ **Aggregations** - Phân tích dữ liệu real-time
- ✅ **Scalable** - Phân tán, replicate tự động

---

## 3. Cache & Message Broker

### 3.1. Redis

**Thông tin cơ bản:**
- **Image:** redis:latest
- **Port:** 6379
- **Container:** redis
- **Password:** 123456789Aa
- **Persistence:** --save 20 1

**Các Service sử dụng:**
| Service | Use Case |
|---------|----------|
| Basket Service | Cache shopping cart |
| All Services | Distributed cache |

**Configuration:**
```json
{
  "RedisCache": {
    "EndPoint": "redis:6379",
    "Password": "123456789Aa",
    "InstanceName": "basket:"
  }
}
```

**Lý do chọn Redis:**
- ✅ **Sub-millisecond Latency** - Phản hồi cực nhanh
- ✅ **Data Structures** - String, Hash, List, Set, Sorted Set
- ✅ **Pub/Sub** - Real-time messaging
- ✅ **Persistence** - Có thể lưu xuống disk

---

### 3.2. RabbitMQ

**Thông tin cơ bản:**
- **Image:** rabbitmq:management
- **AMQP Port:** 5672
- **Management UI:** 15672
- **Metrics:** 15692
- **Container:** rabbitmq

**Các Service sử dụng:**
Tất cả services đều sử dụng RabbitMQ cho:
- Domain Events
- Integration Events
- Outbox/Inbox Pattern

**Configuration:**
```json
{
  "MessageBroker": {
    "Host": "amqp://rabbitmq:5672",
    "UserName": "admin",
    "Password": "123456789Aa",
    "Port": 5672
  }
}
```

**Lý do chọn RabbitMQ:**
- ✅ **Reliable Delivery** - At-least-once delivery guarantee
- ✅ **Flexible Routing** - Exchange, Queue, Binding
- ✅ **Management UI** - Giám sát trực quan
- ✅ **Plugin Support** - Delayed message, tracing, v.v.

---

## 4. Object Storage

### 4.1. MinIO

**Thông tin cơ bản:**
- **Image:** minio/minio:latest
- **API Port:** 9000
- **Console Port:** 9001
- **Container:** minio
- **Access Key:** minioadmin
- **Secret Key:** minioadmin

**Các Service sử dụng:**
| Service | Use Case |
|---------|----------|
| Catalog Service | Product images |
| Order Service | Order attachments |
| Inventory Service | Inventory documents |
| Notification Service | Email attachments |

**Configuration:**
```json
{
  "MinIO": {
    "Endpoint": "minio:9000",
    "AccessKey": "minioadmin",
    "SecretKey": "minioadmin",
    "Secure": false
  }
}
```

**Lý do chọn MinIO:**
- ✅ **S3 Compatible** - API tương thích Amazon S3
- ✅ **High Performance** - Tốc độ đọc/ghi cao
- ✅ **Scalable** - Scale horizontally
- ✅ **Self-hosted** - Không phụ thuộc cloud provider

---

## 5. Observability Stack

### 5.1. Loki (Logging)

**Thông tin cơ bản:**
- **Image:** grafana/loki:latest
- **Port:** 3100
- **Container:** loki

**Chức năng:**
- Aggregated logging từ tất cả services
- Query logs với Grafana
- Label-based indexing

---

### 5.2. Tempo (Tracing)

**Thông tin cơ bản:**
- **Image:** grafana/tempo:latest
- **Port:** 3200
- **Container:** tempo

**Chức năng:**
- Distributed tracing
- Request flow visualization
- Performance bottleneck detection

---

### 5.3. Prometheus (Metrics)

**Thông tin cơ bản:**
- **Image:** prom/prometheus:latest
- **Port:** 9090
- **Container:** prometheus

**Chức năng:**
- Time-series metrics collection
- Alerting rules
- Service health monitoring

---

### 5.4. Grafana (Visualization)

**Thông tin cơ bản:**
- **Image:** grafana/grafana:latest
- **Port:** 3000
- **Container:** grafana

**Chức năng:**
- Unified dashboard cho metrics, logs, traces
- Custom alerting
- Data source: Prometheus, Loki, Tempo

---

## 6. Tổng Hợp Theo Service

| Service | Relational DB | NoSQL | Cache | Object Storage | Message Broker |
|---------|--------------|-------|-------|----------------|----------------|
| Catalog | PostgreSQL | - | - | MinIO | RabbitMQ |
| Basket | - | MongoDB | Redis | - | RabbitMQ |
| Order | SQL Server | - | - | MinIO | RabbitMQ |
| Inventory | MySQL | - | - | MinIO | RabbitMQ |
| Discount | - | MongoDB | - | - | RabbitMQ |
| Report | - | MongoDB | - | - | RabbitMQ |
| Search | - | Elasticsearch | - | - | RabbitMQ |
| Notification | - | MongoDB | - | MinIO | RabbitMQ |
| Communication | - | - | - | - | RabbitMQ |

---

## 7. Kiến Trúc Tổng Thể

```mermaid
flowchart TB
    subgraph "ProgCoder Shop - Storage Architecture"
        direction TB
        
        subgraph "Data Layer"
            PG[(PostgreSQL
            Catalog)]
            SQL[(SQL Server
            Order)]
            MY[(MySQL
            Inventory)]
            MG[(MongoDB
            Basket/Report/Discount)]
            ES[(Elasticsearch
            Search)]
        end
        
        subgraph "Cache & Queue"
            RD[(Redis
            Cache)]
            RMQ[(RabbitMQ
            Message Broker)]
        end
        
        subgraph "Storage"
            MIN[(MinIO
            Object Storage)]
        end
        
        subgraph "Observability"
            LO[(Loki
            Logs)]
            TE[(Tempo
            Traces)]
            PR[(Prometheus
            Metrics)]
            GF[(Grafana
            Dashboard)]
        end
        
        subgraph "Services"
            CAT[Catalog API]
            BAS[Basket API]
            ORD[Order API]
            INV[Inventory API]
            DIS[Discount API]
            REP[Report API]
            SEA[Search API]
            NOT[Notification API]
        end
        
        CAT --> PG
        CAT --> MIN
        BAS --> MG
        BAS --> RD
        ORD --> SQL
        ORD --> MIN
        INV --> MY
        INV --> MIN
        DIS --> MG
        REP --> MG
        SEA --> ES
        NOT --> MG
        NOT --> MIN
        
        CAT --> RMQ
        BAS --> RMQ
        ORD --> RMQ
        INV --> RMQ
        DIS --> RMQ
        REP --> RMQ
        SEA --> RMQ
        NOT --> RMQ
        
        CAT -.-> LO
        CAT -.-> TE
        CAT -.-> PR
    end
```

---

## 8. Best Practices

### 8.1. Connection Management
- Sử dụng Connection Pooling
- Retry policy với exponential backoff
- Circuit breaker pattern

### 8.2. Data Consistency
- **Relational DB:** ACID transactions
- **MongoDB:** Multi-document transactions (v4.0+)
- **Outbox Pattern:** Đảm bảo message delivery
- **Inbox Pattern:** Đảm bảo idempotency

### 8.3. Backup & Recovery
- **PostgreSQL:** pg_dump, WAL archiving
- **SQL Server:** Full/Differential/Log backups
- **MySQL:** mysqldump, binary logs
- **MongoDB:** mongodump, oplog
- **Redis:** RDB snapshots, AOF

### 8.4. Monitoring
- Query performance metrics
- Connection pool usage
- Disk space monitoring
- Replication lag (nếu có)

---

## 9. Port Mapping

| Service | Container | Internal Port | External Port |
|---------|-----------|---------------|---------------|
| PostgreSQL | postgres-sql | 5432 | 5432 |
| SQL Server | sql-server | 1433 | 1433 |
| MySQL | mysql | 3306 | 3306 |
| MongoDB | mongodb | 27017 | 27017 |
| Redis | redis | 6379 | 6379 |
| RabbitMQ | rabbitmq | 5672 | 5672 |
| RabbitMQ Management | rabbitmq | 15672 | 15672 |
| Elasticsearch | elasticsearch | 9200 | 9200 |
| MinIO API | minio | 9000 | 9000 |
| MinIO Console | minio | 9001 | 9001 |
| Loki | loki | 3100 | 3100 |
| Tempo | tempo | 3200 | 3200 |
| Prometheus | prometheus | 9090 | 9090 |
| Grafana | grafana | 3000 | 3000 |

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** AI Assistant - Storage Technologies Documentation
