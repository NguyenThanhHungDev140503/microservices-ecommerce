# AST-Grep MCP Server - Test Usecases

Tài liệu này chứa các usecase thực tế để test khả năng của AST-Grep MCP Server trong dự án **ProgCoder Shop Microservices**.

## Tổng quan về AST-Grep MCP Tools

AST-Grep MCP Server cung cấp 4 tools chính:

| Tool | Mô tả | Khi nào sử dụng |
|------|-------|-----------------|
| `dump_syntax_tree` | Hiển thị cấu trúc AST của code | Debug pattern, hiểu cấu trúc syntax tree |
| `test_match_code_rule` | Test YAML rule với code snippet | Validate rule trước khi áp dụng vào project |
| `find_code` | Tìm code theo pattern đơn giản | Tìm kiếm nhanh các node AST đơn lẻ |
| `find_code_by_rule` | Tìm code theo YAML rule phức tạp | Tìm kiếm nâng cao với nhiều điều kiện |

---

## Usecase 1: Tìm tất cả MassTransit Consumers

### Mục đích
Tìm tất cả các class implement `IConsumer<T>` để review event handlers.

### Pattern đơn giản (find_code)
```
project_folder: /home/nguyen-thanh-hung/Documents/Code/progcoder-shop-microservices
pattern: class $NAME : IConsumer<$EVENT>
language: csharp
```

### YAML Rule nâng cao (find_code_by_rule)
```yaml
id: find-masstransit-consumers
language: csharp
rule:
  kind: class_declaration
  has:
    kind: base_list
    has:
      kind: generic_name
      pattern: IConsumer<$EVENT>
```

### Kết quả mong đợi
Tìm thấy các file như:
- `OrderDeliveredIntegrationEventHandler.cs`
- `OrderCancelledIntegrationEventHandler.cs`
- `BasketCheckoutIntegrationEventHandler.cs`
- `StockChangedEventHandler.cs`

---

## Usecase 2: Tìm tất cả CQRS Commands và Handlers

### Mục đích
Liệt kê tất cả các Command và CommandHandler trong kiến trúc CQRS.

### Pattern cho Commands
```
project_folder: /home/nguyen-thanh-hung/Documents/Code/progcoder-shop-microservices/src
pattern: public record $NAME($$$) : ICommand<$RESULT>
language: csharp
```

### YAML Rule cho CommandHandlers
```yaml
id: find-command-handlers
language: csharp
rule:
  kind: class_declaration
  has:
    kind: base_list
    has:
      pattern: ICommandHandler<$COMMAND, $RESULT>
```

### Kết quả mong đợi
- `CreateProductCommand` / `CreateProductCommandHandler`
- `CreateOrderCommand` / `CreateOrderCommandHandler`
- `DeleteBrandCommand` / `DeleteBrandCommandHandler`

---

## Usecase 3: Tìm các Async Methods không có CancellationToken

### Mục đích
Code review - tìm các async methods thiếu CancellationToken parameter (potential issue).

### YAML Rule
```yaml
id: async-without-cancellation-token
language: csharp
rule:
  kind: method_declaration
  all:
    - has:
        kind: modifier
        pattern: async
    - not:
        has:
          kind: parameter_list
          has:
            kind: parameter
            pattern: CancellationToken $NAME
```

### Lợi ích
Giúp phát hiện các async methods có thể không hỗ trợ cancellation đúng cách.

---

## Usecase 4: Tìm Try-Catch blocks không log exception

### Mục đích
Đảm bảo tất cả các exception đều được log đúng cách.

### YAML Rule
```yaml
id: try-catch-without-logging
language: csharp
rule:
  kind: try_statement
  has:
    kind: catch_clause
    not:
      has:
        kind: invocation_expression
        any:
          - pattern: logger.LogError($$$)
          - pattern: _logger.LogError($$$)
          - pattern: Logger.LogError($$$)
```

---

## Usecase 5: Tìm Entity Classes với Marten Document Store

### Mục đích
Liệt kê tất cả Entities sử dụng Marten cho Event Sourcing.

### Pattern
```
project_folder: /home/nguyen-thanh-hung/Documents/Code/progcoder-shop-microservices/src
pattern: session.Store($ENTITY)
language: csharp
```

### YAML Rule alternative
```yaml
id: find-marten-entities
language: csharp
rule:
  kind: invocation_expression
  pattern: $SESSION.Store($ENTITY)
  inside:
    kind: method_declaration
    stopBy: end
```

---

## Usecase 6: Tìm FluentValidation Rules

### Mục đích
Audit tất cả validation rules trong project.

### YAML Rule
```yaml
id: find-fluent-validators
language: csharp
rule:
  kind: class_declaration
  has:
    kind: base_list
    has:
      pattern: AbstractValidator<$TYPE>
```

### Kết quả mong đợi
- `CreateProductCommandValidator`
- `CreateOrderCommandValidator`
- Và các validator khác...

---

## Usecase 7: Tìm Dependency Injection Registrations

### Mục đích
Review cách DI container được configure.

### Pattern cho AddScoped
```
pattern: services.AddScoped<$INTERFACE, $IMPL>()
language: csharp
```

### Pattern cho AddSingleton
```
pattern: services.AddSingleton<$TYPE>()
language: csharp
```

### YAML Rule tổng hợp
```yaml
id: find-di-registrations
language: csharp
rule:
  kind: invocation_expression
  any:
    - pattern: services.AddScoped<$A, $B>()
    - pattern: services.AddTransient<$A, $B>()
    - pattern: services.AddSingleton<$A>()
    - pattern: services.AddMediatR($$$)
    - pattern: services.AddMassTransit($$$)
```

---

## Usecase 8: Tìm gRPC Services

### Mục đích
Liệt kê tất cả gRPC service implementations.

### YAML Rule
```yaml
id: find-grpc-services
language: csharp
rule:
  kind: class_declaration
  has:
    kind: base_list
    has:
      pattern: $SERVICE.${NAME}Base
```

### Kết quả mong đợi
- `DiscountGrpcService : DiscountProtoService.DiscountProtoServiceBase`
- `ReportGrpcService : ReportProtoService.ReportProtoServiceBase`
- `OrderGrpcService : OrderProtoService.OrderProtoServiceBase`

---

## Usecase 9: Debug Syntax Tree

### Mục đích
Hiểu cấu trúc AST của một đoạn code cụ thể để viết pattern chính xác.

### Sử dụng dump_syntax_tree
```
code: |
  public class MyHandler : IConsumer<OrderCreatedEvent>
  {
      public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
      {
          // handler logic
      }
  }
language: csharp
format: cst
```

### Format options
- `cst` - Concrete Syntax Tree (full detail)
- `ast` - Abstract Syntax Tree (simplified)
- `pattern` - Pattern interpretation

---

## Usecase 10: Tìm Integration Events (Cross-service communication)

### Mục đích
Liệt kê tất cả Integration Events được publish/consume giữa các services.

### YAML Rule cho Publish
```yaml
id: find-published-events
language: csharp
rule:
  kind: invocation_expression
  any:
    - pattern: await $CONTEXT.Publish($EVENT, $$$)
    - pattern: $BUS.Publish($EVENT, $$$)
```

### Pattern cho Event Definitions
```
pattern: public record $NAME($$$) : IntegrationEvent
language: csharp
```

---

## Usecase 11: Tìm Logger Usage Patterns

### Mục đích
Audit logging practices trong project.

### YAML Rule
```yaml
id: find-logger-usage
language: csharp
rule:
  kind: invocation_expression
  any:
    - pattern: logger.LogInformation($$$)
    - pattern: logger.LogWarning($$$)
    - pattern: logger.LogError($$$)
    - pattern: _logger.LogInformation($$$)
    - pattern: _logger.LogWarning($$$)
    - pattern: _logger.LogError($$$)
```

---

## Usecase 12: Tìm Unit of Work Pattern Usage

### Mục đích
Verify proper transaction handling với UnitOfWork pattern.

### YAML Rule
```yaml
id: find-unit-of-work
language: csharp
rule:
  kind: invocation_expression
  any:
    - pattern: unitOfWork.SaveChangesAsync($$$)
    - pattern: _unitOfWork.SaveChangesAsync($$$)
    - pattern: await unitOfWork.SaveChangesAsync($$$)
```

---

## Tips sử dụng AST-Grep hiệu quả

### 1. Sử dụng Meta-variables
- `$NAME` - Match single node  
- `$$$` - Match multiple nodes (0 hoặc nhiều)
- `$$ARGS` - Named multi-match

### 2. Debugging Pattern
1. Dùng `dump_syntax_tree` để xem cấu trúc code thực tế
2. Dùng `test_match_code_rule` để validate rule trước khi chạy
3. Dùng `find_code_by_rule` với `max_results: 5` để test nhanh

### 3. YAML Rule Tips
- Sử dụng `stopBy: end` trong `inside/has` rules để traverse đầy đủ
- Sử dụng `kind` để match chính xác loại node
- Kết hợp `all`, `any`, `not` cho logic phức tạp

### 4. Performance
- Specify `language` để giảm file scan
- Sử dụng `max_results` để giới hạn output
- Target specific folders thay vì toàn bộ project

---

## So sánh với các tool khác

| Feature | ast-grep | grep_search | code-graph-rag |
|---------|----------|-------------|----------------|
| Pattern matching | AST-based | Text-based | Semantic |
| Multi-language | ✅ 20+ languages | ✅ Any text | ✅ Popular languages |
| Refactoring | ✅ | ❌ | ❌ |
| Relationship analysis | ❌ | ❌ | ✅ |
| Speed | Fast | Very Fast | Medium |

### Khi nào dùng ast-grep?
- Cần tìm kiếm structural code patterns
- Muốn match chính xác syntax (không match trong comments/strings)
- Cần refactoring với code transformation
- Viết custom lint rules

