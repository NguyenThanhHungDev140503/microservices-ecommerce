# So Sánh AST-Grep MCP vs Code-Index MCP

> **Ngày đánh giá**: 16/01/2026  
> **Dự án test**: ProgCoder Shop Microservices (C#/.NET)

---

## 1. Tổng Quan

| Tiêu chí | AST-Grep MCP | Code-Index MCP |
|----------|--------------|----------------|
| **Loại tìm kiếm** | Structural (AST-based) | Text-based (grep) |
| **Ngôn ngữ** | 20+ (Tree-sitter) | Any text file |
| **Meta-variables** | ✅ `$NAME`, `$$$` | ❌ Chỉ regex |
| **YAML Rules** | ✅ Có | ❌ Không |
| **Symbol extraction** | ❌ | ✅ Function/Class |
| **Index required** | ❌ | ✅ (tự động) |

---

## 2. Kết Quả So Sánh Chi Tiết

### 2.1. Tìm MassTransit Consumers

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `IConsumer<$EVENT>` | 10 matches | ⭐⭐⭐⭐⭐ Chỉ match interface, có event type |
| **Code-Index** | `IConsumer<` | 10 matches | ⭐⭐⭐⭐⭐ Kết quả tương đương |

**Nhận xét**: Cả hai đều tìm chính xác 10 consumers. AST-Grep có ưu điểm là capture được `$EVENT` type.

---

### 2.2. Tìm CQRS Commands

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `ICommand<$RESULT>` | 46 matches | ⭐⭐⭐⭐⭐ Có return type |
| **Code-Index** | `ICommand<` | 51 matches | ⭐⭐⭐⭐ Nhiều hơn (có cả interface def) |

**Nhận xét**: Code-Index tìm thấy nhiều hơn vì match cả interface definitions, không chỉ implementations.

---

### 2.3. Tìm FluentValidation Validators

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `AbstractValidator<$TYPE>` | 44 matches | ⭐⭐⭐⭐⭐ |
| **Code-Index** | `AbstractValidator<` | 44 matches | ⭐⭐⭐⭐⭐ |

**Nhận xét**: Kết quả hoàn toàn tương đương.

---

### 2.4. Tìm Logger Usage (LogError)

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `logger.LogError($$$)` | 20 matches | ⭐⭐⭐⭐⭐ Match method calls |
| **Code-Index** | `logger.LogError` | 29 matches | ⭐⭐⭐⭐ Match text strings |

**Nhận xét**: Code-Index tìm thấy nhiều hơn vì match cả trong comments và strings nếu có.

---

### 2.5. Tìm MediatR Configuration

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `services.AddMediatR($$$)` | 8 matches | ⭐⭐⭐⭐⭐ Full invocation |
| **Code-Index** | `services.AddMediatR` | 8 matches | ⭐⭐⭐⭐⭐ |

**Nhận xét**: Kết quả hoàn toàn tương đương.

---

### 2.6. Tìm Marten Document Store

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `session.Store($ENTITY)` | 20 matches | ⭐⭐⭐⭐⭐ Có entity name |
| **Code-Index** | `session.Store` | 20 matches | ⭐⭐⭐⭐⭐ |

**Nhận xét**: Kết quả tương đương, AST-Grep có thể capture entity name.

---

### 2.7. Tìm Exception Throwing

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `throw new $EXCEPTION($$$)` | 102 matches | ⭐⭐⭐⭐⭐ Có exception type |
| **Code-Index** | `throw new` | 174 matches | ⭐⭐⭐ Match rộng hơn |

**Nhận xét**: Code-Index match nhiều hơn vì bao gồm cả throw expressions trong lambda, object initializers, etc.

---

### 2.8. Tìm Query Handlers

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `IQueryHandler<$QUERY, $RESULT>` | 41 matches | ⭐⭐⭐⭐⭐ |
| **Code-Index** | `IQueryHandler<` | 42 matches | ⭐⭐⭐⭐⭐ |

**Nhận xét**: Gần như tương đương.

---

### 2.9. Tìm Notification Handlers

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `INotificationHandler<$EVENT>` | 13 matches | ⭐⭐⭐⭐⭐ |
| **Code-Index** | `INotificationHandler<` | 13 matches | ⭐⭐⭐⭐⭐ |

**Nhận xét**: Kết quả hoàn toàn tương đương.

---

### 2.10. Tìm UnitOfWork SaveChanges

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | `await unitOfWork.SaveChangesAsync($$$)` | 29 matches | ⭐⭐⭐⭐⭐ |
| **Code-Index** | `await unitOfWork.SaveChangesAsync` | 29 matches | ⭐⭐⭐⭐⭐ |

**Nhận xét**: Kết quả hoàn toàn tương đương.

---

### 2.11. Tìm gRPC Services

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | YAML rule (base class regex) | 5 matches | ⭐⭐⭐⭐ Chính xác services |
| **Code-Index** | `GrpcService` | 40 matches | ⭐⭐⭐ Match rộng (cả clients) |

**Nhận xét**: AST-Grep chính xác hơn khi tìm service implementations, Code-Index match cả usage.

---

### 2.12. Tìm Try-Catch Blocks

| Tool | Pattern | Kết quả | Đánh giá |
|------|---------|---------|----------|
| **AST-Grep** | YAML rule (kind: try_statement) | 53 matches | ⭐⭐⭐⭐⭐ Full try-catch block |
| **Code-Index** | `try` (failed - unsafe regex) | ❌ Error | ⭐ Không hỗ trợ |

**Nhận xét**: Code-Index không thể tìm kiếm từ khóa ngắn như `try`. AST-Grep hoàn toàn vượt trội cho usecase này.

---

## 3. Bảng Tổng Hợp

| Usecase | AST-Grep | Code-Index | Winner |
|---------|----------|------------|--------|
| IConsumer | 10 | 10 | 🤝 Tie |
| ICommand | 46 | 51 | AST-Grep (chính xác hơn) |
| AbstractValidator | 44 | 44 | 🤝 Tie |
| LogError | 20 | 29 | Tùy mục đích |
| AddMediatR | 8 | 8 | 🤝 Tie |
| session.Store | 20 | 20 | 🤝 Tie |
| throw new | 102 | 174 | Tùy mục đích |
| IQueryHandler | 41 | 42 | 🤝 Tie |
| INotificationHandler | 13 | 13 | 🤝 Tie |
| SaveChangesAsync | 29 | 29 | 🤝 Tie |
| gRPC Services | 5 | 40 | AST-Grep (chính xác) |
| Try-Catch | 53 | ❌ Error | **AST-Grep** |

---

## 4. Điểm Mạnh & Điểm Yếu

### 4.1. AST-Grep MCP

**Điểm mạnh:**
- ✅ Tìm kiếm structural, không match trong comments/strings
- ✅ Meta-variables (`$NAME`, `$$$`) capture values
- ✅ YAML rules cho complex patterns
- ✅ Tìm được syntax structures (try-catch, class declarations)
- ✅ Debug tools (`dump_syntax_tree`)

**Điểm yếu:**
- ❌ Cần hiểu AST structure để viết patterns phức tạp
- ❌ Không có symbol extraction (function list, etc.)
- ❌ Learning curve cao hơn

### 4.2. Code-Index MCP

**Điểm mạnh:**
- ✅ Tìm kiếm text nhanh, đơn giản
- ✅ Tự động index project
- ✅ Symbol extraction (get_file_summary)
- ✅ Find files by pattern (glob)
- ✅ Pagination support

**Điểm yếu:**
- ❌ Match cả trong comments/strings
- ❌ Không có meta-variables
- ❌ Không tìm được syntax structures
- ❌ Một số patterns ngắn bị block (security)

---

## 5. Khuyến Nghị Sử Dụng

### Khi nào dùng AST-Grep?

| Scenario | Lý do |
|----------|-------|
| Tìm code patterns cụ thể | Không match trong comments/strings |
| Audit coding conventions | YAML rules cho complex patterns |
| Refactoring analysis | Capture values với meta-variables |
| Tìm syntax structures | try-catch, class declarations, etc. |

### Khi nào dùng Code-Index?

| Scenario | Lý do |
|----------|-------|
| Tìm kiếm text nhanh | Đơn giản, không cần học syntax |
| Explore codebase | Symbol extraction, file summary |
| Find files by pattern | Glob support |
| Tìm trong mọi loại file | Không giới hạn ngôn ngữ |

### Kết hợp cả hai?

**Workflow đề xuất:**
1. **Code-Index** để explore codebase, tìm file/symbol
2. **AST-Grep** để phân tích patterns chính xác
3. **Code-Graph-RAG** để hiểu relationships

---

## 6. Kết Luận

| Tiêu chí | AST-Grep | Code-Index |
|----------|----------|------------|
| **Độ chính xác** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Tốc độ** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Dễ sử dụng** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Flexibility** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Symbol extraction** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Overall** | **4.5/5** | **4.0/5** |

**Tổng kết**: 
- **AST-Grep** phù hợp cho **code analysis chuyên sâu**, đặc biệt khi cần độ chính xác cao
- **Code-Index** phù hợp cho **tìm kiếm nhanh** và **explore codebase**
- Kết hợp cả hai sẽ tạo ra workflow hiệu quả nhất
