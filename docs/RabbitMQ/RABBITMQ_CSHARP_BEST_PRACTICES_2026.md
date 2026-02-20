# RabbitMQ with C# (.NET) Best Practices Guide (2026 Edition)

Tài liệu này tổng hợp các best practices và patterns hiện đại nhất khi tích hợp RabbitMQ với C# và .NET (v8/9/10) cho năm 2025-2026.

> **Nguồn tham khảo:** RabbitMQ.Client v7, MassTransit, OpenTelemetry, .NET Aspire

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Technology Stack 2026](#2-technology-stack-2026)
3. [Connection & Channel Management](#3--connection--channel-management)
4. [Publisher Patterns](#4-publisher-patterns)
5. [Consumer Patterns](#5-consumer-patterns)
6. [Error Handling & Resilience](#6-error-handling--resilience)
7. [Message Serialization](#7-message-serialization)
8. [Observability & Monitoring](#8-observability--monitoring)
9. [Testing Strategies](#9-testing-strategies)
10. [Deployment & Operations](#10-deployment--operations)
11. [Migration Checklist (v6 → v7)](#11-migration-checklist-v6--v7)
12. [RabbitMQ Streams (Event Sourcing)](#12-rabbitmq-streams-event-sourcing-ready)
13. [Security Best Practices (2026)](#13-security-best-practices-2026)
14. [Native AOT & .NET 9 Considerations](#14-native-aot--net-9-considerations)
15. [Advanced Performance Tuning](#15-advanced-performance-tuning)

---

## 1. Architecture Overview

### Recommended High-Level Architecture

```
┌─────────────────┐
│   Client App   │
│   (ASP.NET)    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│     MassTransit / RabbitMQ.Client v7    │  ← Abstraction Layer
└─────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│           RabbitMQ Cluster               │  ← Quorum Queues + Streams
│  (High Availability + Durability)        │
└─────────────────────────────────────────────┘
```

### Key Principles for 2026

1.  **Async-First:** RabbitMQ.Client v7 loại bỏ hoàn toàn Sync API → Tất cả là `await`
2.  **Quorum-First:** Classic Mirrored Queues đã deprecated → Chỉ dùng **Quorum Queues**
3.  **Resilience-First:** Circuit Breaker + Polly là bắt buộc
4.  **Observability-First:** OpenTelemetry mặc định cho tracing/metrics

---

## 2. Technology Stack 2026

### Recommended Libraries

| Component | Recommendation | Why |
| :--- | :--- | :--- |
| **Abstraction Layer** | **MassTransit v9+** | Enterprise-grade, hỗ trợ Sagas, Outbox, Test Harness |
| **Native Client** | **RabbitMQ.Client v7** | Async-only, hiệu năng cao hơn v6 |
| **Resilience** | **Polly 8+** / **Microsoft.Extensions.Resilience** | Circuit Breaker, Retry, Fallback |
| **Serialization** | **System.Text.Json** (default) / **MessagePack** (performance-critical) | Native, nhanh, hỗ trợ .NET 8 Source Generators |
| **Observability** | **OpenTelemetry.NET** | Distributed tracing, metrics, logs tiêu chuẩn |
| **Health Checks** | **AspNetCore.HealthChecks.RabbitMQ** | Liveness/Readiness probes for K8s |
| **Testing** | **MassTransit.Testing** | In-memory test harness |

### Anti-Pattern Libraries (Avoid)

- ❌ **Brighter v9-** (nếu vẫn dùng RabbitMQ.Client v6 sync) → Cập nhật v10+
- ❌ **NServiceBus** (nếu không cần Enterprise features) → MassTransit nhẹ hơn
- ❌ **Wolverine** (nếu chỉ RabbitMQ) → MassTransit có cộng đồng lớn hơn

---

## 3.  Connection & Channel Management

### 3.1 Connection Lifecycle (RabbitMQ.Client v7)

**Migration Note:** `IModel` → `IChannel`

```csharp
public class RabbitMqConnectionManager : IAsyncDisposable
{
    private readonly IConnectionFactory _factory;
    private readonly SemaphoreSlim _connectionLock = new(1);
    private IConnection? _connection;
    private IChannel? _channel;
    
    public RabbitMqConnectionManager(string connectionString)
    {
        _factory = new ConnectionFactory
        {
            Uri = new Uri(connectionString),
            DispatchConsumersAsync = true, // Quan trọng cho performance
            ConsumerDispatchConcurrency = Environment.ProcessorCount
        };
    }
    
    public async Task<IChannel> GetChannelAsync()
    {
        if (_channel != null && _connection?.IsOpen == true)
            return _channel;
            
        await _connectionLock.WaitAsync();
        try
        {
            if (_channel == null || _connection?.IsOpen != true)
            {
                // V7: CreateConnectionAsync thay vì CreateConnection()
                _connection = await _factory.CreateConnectionAsync();
                _channel = await _connection.CreateChannelAsync();
            }
            return _channel;
        }
        finally
        {
            _connectionLock.Release();
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_channel != null)
            await _channel.DisposeAsync();
        if (_connection != null)
            await _connection.DisposeAsync();
    }
}
```

### 3.2 MassTransit Configuration (Best Practice)

```csharp
// Program.cs - Startup Configuration
builder.Services.AddMassTransit(x =>
{
    x.AddConsumers(Assembly.GetExecutingAssembly());
    
    // Global retry policy - áp dụng cho tất cả consumers
    x.AddConfigureEndpointsCallback((context, name, cfg) =>
    {
        cfg.UseDelayedRedelivery(r => 
            r.Intervals(TimeSpan.FromSeconds(5), TimeSpan.FromMinutes(5)));
        cfg.UseMessageRetry(r => r.Intervals(100, 500, 1000));
        cfg.UseInMemoryOutbox(context);
    });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("guest");
            h.Password("guest");
            h.PublisherConfirmation = true; // Bắt buộc để biết reject-publish
            h.Heartbeat = TimeSpan.FromSeconds(60);
        });
        
        // Cấu hình mặc định cho Quorum Queues
        cfg.SetQuorumQueue();
        
        cfg.ConfigureEndpoints(context);
    });
});
```

### 3.3 Connection Pooling

**NOT Recommended:** Tự implement connection pool.
**Recommended:** MassTransit tự quản lý connections/channels efficiently.

Nếu vẫn phải dùng RabbitMQ.Client trực tiếp:
- Tạo 1 Connection per application instance
- Tạo 1 Channel per thread/consumer
- Channel là thread-safe nhưng heavy-weight

---

## 4. Publisher Patterns

### 4.1 Publisher Confirms (Bắt buộc 2026)

```csharp
public class OrderPublisher
{
    private readonly IBus _bus;
    
    public OrderPublisher(IBus bus)
    {
        _bus = bus;
    }
    
    public async Task PublishOrderCreatedAsync(OrderCreatedEvent evt, CancellationToken ct)
    {
        // MassTransit tự động handle publisher confirms
        // Nếu RabbitMQ reject (queue full + reject-publish), exception sẽ được throw
        try
        {
            await _bus.Publish(evt, ct);
            return true;
        }
        catch (RabbitMQClientException ex) when (ex.IsPublishRejected())
        {
            // Queue full hoặc không thể publish
            _logger.LogWarning("Message rejected: {Reason}", ex.Message);
            await SaveToOutboxAsync(evt); // Fallback
            return false;
        }
    }
}
```

### 4.2 Transactional Outbox Pattern (Anti-Loss Pattern)

Đây là pattern quan trọng nhất để đảm bảo **không mất message**:

```csharp
public class OrderService
{
    private readonly AppDbContext _dbContext;
    private readonly IPublishEndpoint _publishEndpoint;
    private readonly ILogger<OrderService> _logger;

    public async Task CreateOrderAsync(CreateOrderCommand cmd, CancellationToken ct)
    {
        // Bước 1: Lưu vào DB + Outbox trong 1 transaction
        await using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);
        
        try
        {
            var order = new Order(cmd);
            await _dbContext.Orders.AddAsync(order, ct);
            
            // Outbox record sẽ được lưu cùng order
            await _dbContext.SaveChangesAsync(ct);
            
            // MassTransit Outbox sẽ tự động publish message 
            // NHƯNG chỉ khi transaction commit thành công
            await _publishEndpoint.Publish(new OrderCreatedEvent(order.Id), ct);
            
            await transaction.CommitAsync(ct);
        }
        catch
        {
            await transaction.RollbackAsync(ct);
            throw;
        }
    }
}

// Cấu hình EF Core Outbox
services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();
    });
});
```

### 4.3 Retry with Circuit Breaker (Polly)

```csharp
// Định nghĩa Resilience Pipeline
services.AddSingleton<ResiliencePipeline>(sp =>
{
    var logger = sp.GetRequiredService<ILogger<OrderPublisher>>();
    
    return new ResiliencePipelineBuilder()
        // Retry nhanh cho transient errors
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            Delay = TimeSpan.FromSeconds(1),
            OnRetry = args => logger.LogWarning("Retry {Attempt}", args.AttemptNumber)
        })
        // Circuit Breaker để tránh spam khi RabbitMQ sập
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(10),
            BreakDuration = TimeSpan.FromMinutes(2),
            OnBreak = () => logger.LogCritical("Circuit Broken!"),
            OnReset = () => logger.LogInformation("Circuit Reset.")
        })
        .Build();
});
```

---

## 5. Consumer Patterns

### 5.1 Concurrency Control (Quan trọng)

```csharp
// Method 1: Consumer Definition (Recommended)
public class OrderConsumerDefinition : ConsumerDefinition<OrderConsumer>
{
    public OrderConsumerDefinition()
    {
        // EndpointName mặc định = queue name
        EndpointName = "order-service";
        
        // Giới hạn số message được process đồng thời
        // Nên set = số lượng connection pool DB để không làm DB overload
        ConcurrentMessageLimit = 10; 
    }
    
    protected override void ConfigureConsumer(
        IReceiveEndpointConfigurator endpointConfigurator,
        IConsumerConfigurator<OrderConsumer> consumerConfigurator)
    {
        endpointConfigurator.UseMessageRetry(r => r.Interval(3, 1000));
        endpointConfigurator.UseInMemoryOutbox();
    }
}

// Method 2: Manual Configuration (Cho Saga/State Machine)
services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ReceiveEndpoint("order-queue", e =>
        {
            const int ConcurrencyLimit = 20;
            
            // PrefetchCount = số message RabbitMQ push cho consumer mỗi lúc
            // Tương ứng với ConcurrentMessageLimit
            e.PrefetchCount = ConcurrencyLimit;
            
            e.UseMessageRetry(r => r.Interval(5, 1000));
            e.UseInMemoryOutbox();
            
            e.ConfigureConsumer<OrderConsumer>(context);
        });
    });
});
```

### 5.2 Saga Pattern (Distributed Transaction)

```csharp
// Định nghĩa Saga với Concurrency Partitioning
public sealed class OrderStateSagaDefinition : SagaDefinition<OrderState>
{
    private const int ConcurrencyLimit = 20;

    public OrderStateSagaDefinition()
    {
        Endpoint(e =>
        {
            e.Name = "order-saga";
            e.PrefetchCount = ConcurrencyLimit;
        });
    }

    protected override void ConfigureSaga(
        IReceiveEndpointConfigurator endpointConfigurator,
        ISagaConfigurator<OrderState> sagaConfigurator)
    {
        endpointConfigurator.UseMessageRetry(r => r.Interval(5, 1000));
        endpointConfigurator.UseInMemoryOutbox();
        
        // Partitioner đảm bảo chỉ có 1 message được process
        // cho 1 saga instance tại 1 thời điểm
        var partition = endpointConfigurator.CreatePartitioner(ConcurrencyLimit);
        
        sagaConfigurator.Message<SubmitOrder>(x => 
            x.UsePartitioner(partition, m => m.Message.OrderId));
        sagaConfigurator.Message<OrderAccepted>(x => 
            x.UsePartitioner(partition, m => m.Message.OrderId));
    }
}
```

### 5.3 Acknowledgment Patterns

**Recommended:** Sử dụng MassTransit - nó tự động handle Ack/Nack

Nếu dùng RabbitMQ.Client trực tiếp:

```csharp
public class AsyncConsumer : DefaultAsyncConsumer
{
    protected override async Task OnMessageReceivedAsync(
        BasicDeliverEventArgs ea, CancellationToken cancellationToken)
    {
        try
        {
            var message = JsonSerializer.Deserialize<Order>(ea.Body.Span);
            
            await ProcessOrderAsync(message, cancellationToken);
            
            // Ack = xử lý thành công
            await Channel!.BasicAckAsync(ea.DeliveryTag, false, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Consumer error");
            
            // Nack = retry (requeue=true)
            await Channel!.BasicNackAsync(ea.DeliveryTag, false, true, cancellationToken);
        }
    }
}
```

---

## 6. Error Handling & Resilience

### 6.1 Message Retry vs Redelivery

| Strategy | Use Case | Interval |
| :--- | :--- | :--- |
| **Immediate Retry** | Transient errors (DB deadlock) | 100ms, 200ms, 500ms |
| **Delayed Redelivery** | Resource limit (API rate limit) | 5min, 15min, 30min |

```csharp
cfg.UseDelayedRedelivery(r => 
    r.Intervals(
        TimeSpan.FromMinutes(5),   // Retry 1
        TimeSpan.FromMinutes(15),  // Retry 2
        TimeSpan.FromMinutes(30)   // Retry 3
    ));
```

### 6.2 Dead Letter Exchange (DLX) Configuration

```csharp
// Server-side Policy (RabbitMQ Management UI)
{
  "definition": {
    "x-dead-letter-exchange": "dlx-order-error",
    "x-dead-letter-routing-key": "order-error",
    "x-max-length": 100000,
    "x-overflow": "reject-publish", // Khuyến nghị: reject-publish
    "x-delivery-limit": 10          // Tránh infinite loop
  }
}
```

**Note:** Với `overflow=drop-head`, message bị xóa sẽ **KHÔNG** được gửi đến DLX.

### 6.3 Poison Message Handling

```csharp
// Consumer xử lý poison message
public class ErrorConsumer : IConsumer<Fault<SubmitOrder>>
{
    public async Task Consume(ConsumeContext<Fault<SubmitOrder>> context)
    {
        var failedMessage = context.Message.Message;
        var exceptions = context.Message.Exceptions;
        
        _logger.LogError("Order failed: {OrderId}, Exceptions: {Ex}", 
            failedMessage.OrderId, exceptions.Count);
            
        // Lưu vào DB để admin review
        await _errorLogRepository.SaveAsync(new ErrorLog
        {
            OrderId = failedMessage.OrderId,
            Exceptions = exceptions,
            Timestamp = DateTime.UtcNow
        });
    }
}
```

---

## 7. Message Serialization

### 7.1 System.Text.Json (Default cho 2026)

```csharp
// Cấu hình MassTransit dùng System.Text.Json
services.AddMassTransit(x =>
{
    x.AddConfigureEndpointsCallback((context, name, cfg) =>
    {
        cfg.UseMessageJson(SystemTextJsonMessageSerializer.Options);
    });
});

// Hoặc cấu hình custom JsonSerializerOptions
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = false,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};

cfg.UseMessageJson(options);
```

**Pros:** Native .NET 8, tối ưu với Source Generators.

### 7.2 MessagePack (High-Performance Scenario)

```csharp
// Install: MassTransit.MessagePack
services.AddMassTransit(x =>
{
    x.AddConfigureEndpointsCallback((context, name, cfg) =>
    {
        cfg.UseMessagePack(new MessagePackFormatterOptions
        {
            EnableCompression = true
        });
    });
});
```

**Use case:** Telemetry, Metrics, Event Sourcing với throughput cực cao.

---

## 8. Observability & Monitoring

### 8.1 OpenTelemetry Integration

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("OrderService"))
    .AddTracing(tracing =>
    {
        // MassTransit tracing tự động
        tracing.AddMassTransitInstrumentation();
        tracing.AddAspNetCoreInstrumentation();
        tracing.AddHttpClientInstrumentation();
    })
    .AddMetrics(metrics =>
    {
        metrics.AddMassTransitInstrumentation();
        metrics.AddAspNetCoreInstrumentation();
    })
    .AddOtlpExporter(); // OpenTelemetry Protocol

// Prometheus scraping endpoint
builder.MapPrometheusScrapingEndpoint();
```

### 8.2 Health Checks (Kubernetes Ready)

```csharp
builder.Services.AddHealthChecks()
    .AddRabbitMQ(connectionString: "amqp://guest:guest@localhost:5672", 
        name: "rabbitmq")
    .AddDbContext<AppDbContext>(name: "database")
    .AddCheck<MemoryHealthCheck>("memory")
    .AddCheck<DiskHealthCheck>("disk");

// Configure Health Checks UI
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(15);
    options.MaximumHistoryEntriesPerEndpoint(50);
});

app.MapHealthChecks("/health"); // Full health
app.MapHealthChecks("/ready"); // Readiness probe (tag: "ready")
app.MapHealthChecksUI();
```

**Kubernetes Deployment:**

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
```

### 8.3 Critical Metrics to Monitor

| Metric | Alert Threshold | Action |
| :--- | :--- | :--- |
| `rabbitmq_queue_messages_ready` | > 10,000 | Scale consumers |
| `rabbitmq_queue_messages_unacked` | > 1000 | Check consumer deadlock |
| `rabbitmq_channel_open` | Spike | Check connection leak |
| `rabbitmq_consumers_total` | Drop to 0 | Service restart required |
| `rabbitmq_disk_free` | < 5GB | Critical - RabbitMQ may block |

---

## 9. Testing Strategies

### 9.1 Unit Tests with In-Memory Harness

```csharp
using MassTransit.Testing;
using Xunit;

public class OrderConsumerTests
{
    [Fact]
    public async Task Should_Consume_OrderCreated_And_Publish_OrderProcessed()
    {
        // Arrange
        await using var provider = new ServiceCollection()
            .AddMassTransitTestHarness(x =>
            {
                x.AddConsumer<OrderConsumer>();
            })
            .BuildServiceProvider(true);

        var harness = provider.GetRequiredService<ITestHarness>();
        await harness.Start();

        // Act
        await harness.Bus.Publish(new OrderCreated
        {
            OrderId = Guid.NewGuid(),
            CustomerEmail = "test@example.com"
        });

        // Assert
        Assert.True(await harness.Consumed.Any<OrderCreated>());
        Assert.True(await harness.Published.Any<OrderProcessed>());

        var consumerHarness = harness.GetConsumerHarness<OrderConsumer>();
        Assert.True(await consumerHarness.Consumed.Any<OrderCreated>());
    }
}
```

### 9.2 Integration Tests with Testcontainers

```csharp
using Testcontainers.RabbitMq;

public class OrderIntegrationTests : IClassFixture<RabbitMqContainer>
{
    private readonly RabbitMqContainer _rabbitMq;

    public OrderIntegrationTests(RabbitMqContainer rabbitMq)
    {
        _rabbitMq = rabbitMq;
    }

    [Fact]
    public async Task Should_Send_And_Receive_Message()
    {
        // Arrange
        var connectionString = $"amqp://guest:guest@{_rabbitMq.Hostname}:{_rabbitMq.GetMappedPublicPort(5672)}";
        
        var services = new ServiceCollection();
        services.AddMassTransit(x =>
        {
            x.UsingRabbitMq((context, cfg) =>
            {
                cfg.Host(connectionString);
                cfg.ReceiveEndpoint("test-queue", e =>
                {
                    e.ConfigureConsumer<OrderConsumer>(context);
                });
            });
        });

        // Act & Assert...
    }
}
```

---

## 10. Deployment & Operations

### 10.1 Dockerfile (2026 Best Practices)

```dockerfile
# Multi-stage build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app

# Runtime image
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080

# Non-root user (security)
USER app
ENTRYPOINT ["dotnet", "OrderService.dll"]
```

### 10.2 Kubernetes Deployment (Graceful Shutdown)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      terminationGracePeriodSeconds: 30  # Graceful shutdown time
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: RabbitMQ__ConnectionString
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secrets
              key: connection-string
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### 10.3 Graceful Shutdown in .NET

```csharp
// Program.cs
builder.Host.ConfigureServices((hostContext, services) =>
{
    services.Configure<HostOptions>(options =>
    {
        // Đợi 30 giây trước khi force shutdown
        options.ShutdownTimeout = TimeSpan.FromSeconds(30);
    });
});

// MassTransit tự động graceful shutdown khi application shutdown
// - Stop nhận message mới
// - Đợi các messages đang xử lý hoàn thành
// - Close connections
```

---

## 11. Migration Checklist (v6 → v7)

### Breaking Changes in RabbitMQ.Client v7

| Old (v6) | New (v7) | Action Required |
| :--- | :--- | :--- |
| `IModel` | `IChannel` | Rename all `IModel` → `IChannel` |
| `CreateModel()` | `CreateChannelAsync()` | Add `await` |
| `BasicPublish()` | `BasicPublishAsync()` | All methods are now async |
| `connection.Close()` | `await connection.CloseAsync()` | Async dispose |
| `CreateBasicProperties()` | `new BasicProperties()` | Direct instantiation |

### Code Migration Example

```csharp
// ❌ Old (v6)
using IConnection connection = factory.CreateConnection();
using IModel channel = connection.CreateModel();
channel.BasicPublish(exchange, routingKey, body: messageBody);

// ✅ New (v7)
await using IConnection connection = await factory.CreateConnectionAsync();
await using IChannel channel = await connection.CreateChannelAsync();
await channel.BasicPublishAsync(exchange, routingKey, body: messageBody);
```

### Additional Notes

1.  **ReadOnlyMemory<byte> Ownership:** Trong v7, message body memory chỉ valid trong callback handler. Nếu cần lưu lại, phải copy.
2.  **No Blocking Operations:** Tất cả blocking operations sẽ throw `InvalidOperationException`.
3.  **Connection String:** Support cả `amqp://` và `amqps://` (TLS) URI format.

---

## 12. RabbitMQ Streams (Event Sourcing Ready)

### 12.1 Tổng quan Streams

RabbitMQ Streams là feature mới cho phép lưu trữ message dạng append-only log, phù hợp cho Event Sourcing và high-throughput scenarios.

```csharp
// RabbitMQ Stream Client - cài đặt qua NuGet: RabbitMQ.Stream.Client
using RabbitMQ.Stream.Client;
using RabbitMQ.Stream.Client.Reliable;

public class StreamProducerService : IAsyncDisposable
{
    private readonly StreamSystem _system;
    private readonly Producer _producer;
    private readonly string _streamName;
    
    public async Task InitializeAsync()
    {
        _system = await StreamSystem.Create(new StreamSystemConfig
        {
            Endpoints = new List<EndPoint> { new DnsEndPoint("localhost", 5552) },
            UserName = "guest",
            Password = "guest"
        });
        
        // Tạo stream với retention policy
        await _system.CreateStream(new StreamSpec(_streamName)
        {
            MaxLengthBytes = 5_000_000_000, // 5GB max
            MaxSegmentSizeBytes = 500_000_000,
            MaxAge = TimeSpan.FromDays(7)
        });
        
        _producer = await Producer.Create(new ProducerConfig(_system, _streamName)
        {
            Reference = "order-producer", // Cho deduplication
            ConfirmHandler = async confirmation =>
            {
                switch (confirmation.Status)
                {
                    case ConfirmationStatus.Confirmed:
                        _logger.LogDebug("Message confirmed: {Id}", confirmation.PublishingId);
                        break;
                    case ConfirmationStatus.ClientTimeoutError:
                        _logger.LogWarning("Timeout for message: {Id}", confirmation.PublishingId);
                        break;
                }
            }
        });
    }
    
    public async Task PublishEventAsync<T>(T evt) where T : class
    {
        var body = JsonSerializer.SerializeToUtf8Bytes(evt);
        await _producer.Send(new Message(body));
    }
    
    public async ValueTask DisposeAsync()
    {
        await _producer.Close();
        await _system.Close();
    }
}
```

### 12.2 Stream Consumer Pattern

```csharp
public class StreamConsumerService : BackgroundService
{
    private readonly ILogger<StreamConsumerService> _logger;
    private StreamSystem _system;
    private Consumer _consumer;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _system = await StreamSystem.Create(new StreamSystemConfig
        {
            Endpoints = new List<EndPoint> { new DnsEndPoint("localhost", 5552) }
        });
        
        _consumer = await Consumer.Create(new ConsumerConfig(_system, "order-events")
        {
            // Single Active Consumer pattern
            IsSingleActiveConsumer = true,
            Reference = "order-consumer-group",
            
            // Bắt đầu từ offset cụ thể hoặc từ đầu
            OffsetSpec = new OffsetTypeFirst(),
            
            MessageHandler = async (consumer, context, message) =>
            {
                try
                {
                    var evt = JsonSerializer.Deserialize<OrderEvent>(message.Data.Contents);
                    await ProcessEventAsync(evt);
                    
                    // Store offset để resume sau restart
                    await consumer.StoreOffset(context.Offset);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing stream message");
                    // Stream không có nack - cần xử lý error manually
                }
            }
        });
        
        await stoppingToken.WaitHandle.WaitOneAsync();
    }
}
```

### 12.3 Best Practices cho Streams

| Aspect | Best Practice |
| :--- | :--- |
| **Retention** | Luôn định nghĩa retention policy (max length/age) |
| **Deduplication** | Sử dụng producer reference name |
| **Consumer** | Single Active Consumer cho ordered processing |
| **Offset** | Store offset sau mỗi batch, không từng message |
| **Connection** | Long-lived connections, 1 producer/consumer per connection |

---

## 13. Security Best Practices (2026)

### 13.1 TLS/mTLS Configuration

**Bắt buộc cho Production:** TLS 1.2+ và mTLS authentication.

```csharp
// RabbitMQ.Client v7 với TLS
var factory = new ConnectionFactory
{
    Uri = new Uri("amqps://localhost:5671"),
    Ssl = new SslOption
    {
        Enabled = true,
        ServerName = "rabbitmq.internal",
        Version = SslProtocols.Tls12 | SslProtocols.Tls13,
        
        // mTLS: Client certificate
        CertPath = "/certs/client.pfx",
        CertPassphrase = Environment.GetEnvironmentVariable("CERT_PASSWORD"),
        
        // Verify server certificate
        CertificateValidationCallback = (sender, cert, chain, errors) =>
        {
            if (errors == SslPolicyErrors.None) return true;
            
            // Log và reject invalid certificates
            _logger.LogWarning("Certificate error: {Errors}", errors);
            return false;
        }
    }
};
```

### 13.2 MassTransit với TLS

```csharp
services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq.internal", 5671, "/", h =>
        {
            h.Username("service-account");
            h.Password(Environment.GetEnvironmentVariable("RABBITMQ_PASSWORD"));
            
            // Enable TLS
            h.UseSsl(ssl =>
            {
                ssl.Protocol = SslProtocols.Tls12;
                ssl.ServerName = "rabbitmq.internal";
                
                // mTLS client certificate
                ssl.Certificate = new X509Certificate2(
                    "/certs/client.pfx",
                    Environment.GetEnvironmentVariable("CERT_PASSWORD"));
                
                ssl.CertificateValidationCallback = 
                    CertificateValidationCallback; // Custom validation
            });
            
            h.PublisherConfirmation = true;
        });
        
        cfg.ConfigureEndpoints(context);
    });
});
```

### 13.3 Security Checklist

| Item | Requirement |
| :--- | :--- |
| **TLS Version** | TLS 1.2 hoặc 1.3 (disable TLS 1.0/1.1) |
| **Authentication** | mTLS hoặc OAuth 2.0 (không dùng basic auth cho production) |
| **Certificate** | Short-lived certs với auto-rotation |
| **Network** | VPC với firewall, không expose public |
| **Audit** | Enable audit logging cho security events |
| **Updates** | RabbitMQ 4.0.8+ (fix CVE-2025-50200) |

---

## 14. Native AOT & .NET 9 Considerations

### 14.1 Hiện trạng Native AOT với MassTransit

> **Warning:** Native AOT chưa fully supported cho MassTransit do sử dụng reflection-based serialization.

```csharp
// Lỗi khi dùng Native AOT:
// System.InvalidOperationException: Reflection-based serialization has been disabled

// Workaround: Sử dụng Source Generators cho serialization
[JsonSerializable(typeof(OrderCreatedEvent))]
[JsonSerializable(typeof(OrderProcessedEvent))]
public partial class MessagingJsonContext : JsonSerializerContext { }

// Cấu hình MassTransit với custom serializer
services.AddMassTransit(x =>
{
    x.AddConfigureEndpointsCallback((context, name, cfg) =>
    {
        cfg.UseRawJsonSerializer(options =>
        {
            options.TypeInfoResolver = MessagingJsonContext.Default;
        });
    });
});
```

### 14.2 .NET 9 Features với RabbitMQ

```csharp
// .NET 9: Server GC Configuration cho high throughput
<PropertyGroup>
    <ServerGarbageCollection>true</ServerGarbageCollection>
    <GarbageCollectionLargeObjectHeapCompactionMode>
        CompactOnce
    </GarbageCollectionLargeObjectHeapCompactionMode>
</PropertyGroup>

// .NET 9: Improved async patterns với ConfigureAwait
public async Task PublishAsync<T>(T message, CancellationToken ct) where T : class
{
    await _bus.Publish(message, ct).ConfigureAwait(ConfigureAwaitOptions.None);
}
```

### 14.3 Migration Path cho Native AOT

| Step | Action |
| :--- | :--- |
| 1 | Migrate tất cả message types sang record types |
| 2 | Tạo JsonSerializerContext cho tất cả message types |
| 3 | Test với PublishTrimmed=true trước khi AOT |
| 4 | Theo dõi MassTransit roadmap cho full AOT support |

---

## 15. Advanced Performance Tuning

### 15.1 Connection Optimization

```csharp
// Cấu hình tối ưu cho high-throughput
var factory = new ConnectionFactory
{
    Uri = new Uri("amqp://localhost"),
    
    // Tách connection cho publish/consume
    // Tránh back pressure từ consumer acks ảnh hưởng publisher
    ClientProvidedName = "publisher-connection",
    
    // Tuning
    RequestedHeartbeat = TimeSpan.FromSeconds(30),
    RequestedChannelMax = 2047,
    SocketReadTimeout = TimeSpan.FromSeconds(30),
    SocketWriteTimeout = TimeSpan.FromSeconds(30),
    
    // Async dispatch
    DispatchConsumersAsync = true,
    ConsumerDispatchConcurrency = Environment.ProcessorCount * 2
};
```

### 15.2 Prefetch Tuning

```csharp
// Công thức: PrefetchCount = Target throughput × Average processing time
// Ví dụ: 1000 msg/s × 0.02s = 20 prefetch

services.AddMassTransit(x =>
{
    x.AddConsumer<OrderConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost");
        
        cfg.ReceiveEndpoint("high-throughput-queue", e =>
        {
            // Prefetch cao cho throughput-first
            e.PrefetchCount = 50;
            
            // Hoặc prefetch thấp cho latency-first
            // e.PrefetchCount = 1;
            
            e.ConfigureConsumer<OrderConsumer>(context);
        });
    });
});
```

### 15.3 Batch Publishing

```csharp
// MassTransit batch publishing
services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", h =>
        {
            // Enable batch mode cho high throughput
            h.ConfigureBatchPublish(batch =>
            {
                batch.Enabled = true;
                batch.MessageLimit = 100;
                batch.SizeLimit = 65536;
                batch.Timeout = TimeSpan.FromMilliseconds(50);
            });
        });
    });
});

// Disable cho low-latency scenarios
cfg.Host("localhost", h =>
{
    h.ConfigureBatchPublish(batch => batch.Enabled = false);
});
```

### 15.4 Performance Metrics to Monitor

| Metric | Target | Action if Exceeded |
| :--- | :--- | :--- |
| `messages_ready` | < 10,000 | Scale consumers hoặc tăng prefetch |
| `messages_unacked` | < prefetch × consumers | Check consumer processing time |
| `publish_rate` / `deliver_rate` | Balanced | Adjust producer/consumer ratio |
| `memory_used` | < 75% limit | Increase memory hoặc add nodes |
| `disk_free` | > 10GB | RabbitMQ memory alarm at 5GB |

---

## 16. Summary: 2026 Key Takeaways

| Aspect | 2026 Best Practice |
| :--- | :--- |
| **Client Library** | MassTransit (abstraction) hoặc RabbitMQ.Client v7 (native) |
| **Queue Type** | Quorum Queues (mandatory) |
| **Publishing** | Publisher Confirms ON + Circuit Breaker + Batch Publishing |
| **Consuming** | Concurrency limit + Prefetch tuning + Single Active Consumer |
| **Error Handling** | Immediate Retry + Delayed Redelivery + DLX |
| **Reliability** | Transactional Outbox (EF Core) |
| **Observability** | OpenTelemetry + Health Checks + Prometheus |
| **Testing** | In-Memory Test Harness (Unit) + Testcontainers (Integration) |
| **Deployment** | Kubernetes + Graceful Shutdown (30s) |
| **Streams** | Event Sourcing với retention policy + deduplication |
| **Security** | TLS 1.2+/mTLS + OAuth 2.0 + RabbitMQ 4.0.8+ |
| **Performance** | Separate pub/sub connections + Batch mode + Server GC |

---

## References & Further Reading

1.  [RabbitMQ.Client v7 Migration Guide](https://github.com/rabbitmq/rabbitmq-dotnet-client/blob/main/v7-MIGRATION.md)
2.  [MassTransit Documentation](https://masstransit-project.com/)
3.  [.NET Aspire Distributed Systems](https://learn.microsoft.com/en-us/dotnet/aspire/)
4.  [OpenTelemetry .NET](https://opentelemetry.io/docs/instrumentation/net/)
5.  [RabbitMQ Quorum Queues Best Practices](https://www.rabbitmq.com/quorum-queues)
6.  [RabbitMQ Streams .NET Client](https://github.com/rabbitmq/rabbitmq-stream-dotnet-client)
7.  [RabbitMQ TLS Configuration](https://www.rabbitmq.com/docs/ssl)
8.  [Native AOT with .NET 9](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)
9.  [MassTransit Performance Tuning](https://masstransit.io/documentation/configuration/performance)

---

**Version:** 2026.2
**Last Updated:** January 2026 (Updated with Streams, Security, AOT, Performance sections)
**Maintained by:** Architecture Team - Company Name

---

> **Nguồn Research:** Thông tin được tổng hợp từ grep.app (GitHub code search), Tavily web search, và Context7 MCP (MassTransit Documentation).
