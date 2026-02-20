# Báo Cáo Đánh Giá Skill grep.app-search

**Ngày đánh giá:** 2026-01-16  
**Phiên bản skill:** grep-app-search v1.0  
**Dự án test:** progcoder-shop-microservices

---

## 1. Tổng Quan

Skill `grep.app-search` cho phép tìm kiếm code trên GitHub thông qua API của [grep.app](https://grep.app), một công cụ index hàng triệu repository công khai. Skill này đặc biệt hữu ích để:

- Tìm kiếm các pattern implementation phổ biến
- Học hỏi cách các dự án lớn triển khai tính năng
- So sánh và đánh giá các approach khác nhau

---

## 2. Kết Quả Test Use Cases

### 2.1. Use Case: Tìm Implementation MassTransit

```bash
python3 grep_search.py "MassTransit" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 8,713 |
| Thời gian query | 181ms |
| Top repo | MassTransit/MassTransit (5,487) |

**Đánh giá:** ✅ Xuất sắc - Tìm thấy source chính thức và các dự án sử dụng MassTransit

---

### 2.2. Use Case: Tìm IConsumer Pattern

```bash
python3 grep_search.py "IConsumer" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 2,938 |
| Thời gian query | 189ms |
| Top repos | MassTransit (455), SlimMessageBus (120), KafkaFlow (98) |

**Đánh giá:** ✅ Tốt - Tìm được nhiều implementation patterns từ các thư viện nổi tiếng

---

### 2.3. Use Case: Tìm IRequestHandler (MediatR/CQRS)

```bash
python3 grep_search.py "IRequestHandler" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 10,185 |
| Thời gian query | 180ms |
| Top repos | ErsatzTV (346), grandnode2 (269) |

**Đánh giá:** ✅ Xuất sắc - Rất nhiều examples về CQRS pattern

---

### 2.4. Use Case: Tìm gRPC Service

```bash
python3 grep_search.py "gRPC service" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 191 |
| Thời gian query | 128ms |
| Top repos | grpc-dotnet (17), practical-aspnetcore (9) |

**Đánh giá:** ✅ Tốt - Tìm được official examples từ grpc-dotnet và docs

---

### 2.5. Use Case: Tìm RabbitMQ Consumer

```bash
python3 grep_search.py "RabbitMQ Consumer" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 26 |
| Thời gian query | 128ms |
| Top repos | CAP (2), eShop (2), Blog.Core (2) |

**Đánh giá:** ⚠️ Trung bình - Kết quả ít do query dài, nên dùng pattern ngắn hơn

---

### 2.6. Use Case: Regex Search - Exception Classes

```bash
python3 grep_search.py "class.*Exception" --lang "C#" --regex --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 50 |
| Thời gian query | 119ms |
| Top repos | openapi-generator (11), semantic-kernel (11) |

**Đánh giá:** ⚠️ Trung bình - Regex hoạt động nhưng C# không phải top language cho regex này

---

### 2.7. Use Case: Tìm AddDbContext Pattern

```bash
python3 grep_search.py "AddDbContext" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 5,390 |
| Thời gian query | 245ms |
| Top repos | AspNetCore.Docs (296), efcore (42) |

**Đánh giá:** ✅ Xuất sắc - Tìm được official docs và best practices từ Microsoft

---

### 2.8. Use Case: Tìm Saga Pattern

```bash
python3 grep_search.py "Saga pattern" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 4 |
| Thời gian query | 131ms |
| Top repos | CAP (2), Durable-Task-Scheduler (1) |

**Đánh giá:** ⚠️ Ít kết quả - Nên dùng term cụ thể hơn như "ISaga" hoặc "StateMachine"

---

### 2.9. Use Case: Tìm Polly Retry Pattern

```bash
python3 grep_search.py "Polly retry" --lang "C#" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 87 |
| Thời gian query | 133ms |
| Top repos | openapi-generator (67) |

**Đánh giá:** ✅ Tốt - Tìm được các RetryConfiguration examples

---

### 2.10. Use Case: Path Filter (Test Files)

```bash
python3 grep_search.py "GetAwaiter" --lang "C#" --path "test" --max 5
```

| Chỉ số | Kết quả |
|--------|---------|
| Tổng kết quả | 0 |
| Thời gian query | 18ms |

**Đánh giá:** ❌ Không tìm thấy - Path filter có thể quá strict

---

## 3. Tổng Hợp Đánh Giá

### 3.1. Điểm Mạnh

| Đặc điểm | Đánh giá |
|----------|----------|
| **Tốc độ** | ✅ Rất nhanh (100-250ms) |
| **Độ phủ** | ✅ Index hàng triệu repos |
| **Language filter** | ✅ Hoạt động chính xác |
| **Top repos** | ✅ Hiển thị repos uy tín |
| **Snippets** | ✅ Có preview code |

### 3.2. Điểm Yếu

| Đặc điểm | Đánh giá |
|----------|----------|
| **Path filter** | ⚠️ Kết quả không ổn định |
| **Multi-word queries** | ⚠️ Kết quả ít hơn mong đợi |
| **Regex phức tạp** | ⚠️ Có thể không match như ý |
| **Rate limiting** | ⚠️ Có giới hạn (HTTP 429) |

### 3.3. Điểm Tổng Thể

| Tiêu chí | Điểm (1-5) |
|----------|------------|
| Độ tin cậy | ⭐⭐⭐⭐ |
| Tính hữu ích | ⭐⭐⭐⭐⭐ |
| Dễ sử dụng | ⭐⭐⭐⭐ |
| Hiệu năng | ⭐⭐⭐⭐⭐ |
| **Tổng** | **4.25/5** |

---

## 4. Hướng Dẫn Sử Dụng Chi Tiết

### 4.1. Cài Đặt Dependencies

```bash
pip install aiohttp
```

### 4.2. Sử Dụng Command Line

```bash
# Cú pháp cơ bản
python3 grep_search.py "<query>" [options]

# Các options
--lang, -l    # Filter theo ngôn ngữ (Python, C#, JavaScript...)
--repo, -r    # Filter theo repository (owner/repo)
--path, -p    # Filter theo đường dẫn (src/, tests/)
--regex       # Bật chế độ regex
--case        # Case-sensitive search
--max, -m     # Số kết quả tối đa (default: 10)
--no-snippet  # Ẩn code preview
--json        # Output JSON thô
```

### 4.3. Các Ví Dụ Thực Tế

#### Tìm cách implement Consumer trong MassTransit
```bash
python3 grep_search.py "IConsumer<" --lang "C#" --repo "MassTransit/MassTransit"
```

#### Tìm cách setup DbContext với pooling
```bash
python3 grep_search.py "AddDbContextPool" --lang "C#"
```

#### Tìm async/await patterns trong tests
```bash
python3 grep_search.py "async Task" --lang "C#" --path "test"
```

#### Tìm Singleton pattern bằng regex
```bash
python3 grep_search.py "__new__.*cls.*instance" --regex --lang Python
```

### 4.4. Sử Dụng API Trực Tiếp (curl)

```bash
# Cơ bản
curl -s "https://grep.app/api/search?q=FastAPI"

# Với filters
curl -s "https://grep.app/api/search?q=IConsumer&f.lang=C%23&f.repo=MassTransit/MassTransit"

# Regex search
curl -s "https://grep.app/api/search?q=class.*Service&regexp=true&f.lang=C%23"
```

### 4.5. Sử Dụng Trong Python Code

```python
import asyncio
import aiohttp

async def search_github_code(query: str, lang: str = None) -> dict:
    params = {"q": query}
    if lang:
        params["f.lang"] = lang
    
    async with aiohttp.ClientSession() as session:
        async with session.get(
            "https://grep.app/api/search",
            params=params
        ) as response:
            return await response.json()

# Sử dụng
result = asyncio.run(search_github_code("MassTransit", "C#"))
print(f"Found: {result['hits']['total']} results")
```

---

## 5. Best Practices

### 5.1. Query Hiệu Quả

| Nên | Không nên |
|-----|-----------|
| `IConsumer<` | `IConsumer interface implementation` |
| `AddDbContext` | `how to add database context` |
| `async Task` | `asynchronous task pattern` |

### 5.2. Kết Hợp Filters

1. **Bắt đầu rộng** → Không dùng filter
2. **Thu hẹp dần** → Thêm `--lang`
3. **Cụ thể hơn** → Thêm `--repo` hoặc `--path`

### 5.3. Xử Lý Rate Limiting

```python
import time

def search_with_retry(query, max_retries=3):
    for attempt in range(max_retries):
        result = search(query)
        if "error" not in result:
            return result
        if "429" in str(result.get("error", "")):
            time.sleep(60)  # Wait 1 minute
        else:
            break
    return result
```

---

## 6. So Sánh Với Các Công Cụ Khác

| Công cụ | Ưu điểm | Nhược điểm |
|---------|---------|------------|
| **grep.app** | Nhanh, index rộng, miễn phí | Rate limit, chỉ public repos |
| **GitHub Code Search** | Official, private repos | Cần auth, chậm hơn |
| **Sourcegraph** | Rất mạnh, regex | Cần setup, paid |
| **ast-grep** | Structural search | Chỉ local |

---

## 7. Use Cases Phù Hợp Cho Dự Án Microservices

### 7.1. Tìm Best Practices

| Mục đích | Query |
|----------|-------|
| MassTransit consumer | `IConsumer<` --lang C# |
| CQRS handler | `IRequestHandler<` --lang C# |
| gRPC service | `MapGrpcService` --lang C# |
| EF Core setup | `AddDbContext<` --lang C# |
| Polly resilience | `AddPolicyHandler` --lang C# |

### 7.2. Học Từ Dự Án Uy Tín

| Dự án | Query |
|-------|-------|
| eShop | `--repo dotnet/eShop` |
| MassTransit | `--repo MassTransit/MassTransit` |
| Clean Architecture | `--repo ardalis/CleanArchitecture` |

### 7.3. Tìm Error Handling Patterns

```bash
python3 grep_search.py "catch.*Exception" --regex --lang "C#" --repo "dotnet/eShop"
```

---

## 8. Kết Luận

Skill `grep.app-search` là công cụ **rất hữu ích** cho việc:

1. ✅ Tìm implementation examples từ các dự án uy tín
2. ✅ Học best practices nhanh chóng
3. ✅ So sánh các approach khác nhau
4. ⚠️ Nên kết hợp với các công cụ khác (ast-grep, code-graph-rag) cho phân tích sâu hơn

**Khuyến nghị:** Sử dụng như bước đầu tiên khi cần tìm hiểu cách implement một pattern mới trong dự án microservices.

---

## 9. Tài Liệu Tham Khảo

- [grep.app Website](https://grep.app)
- [Skill SKILL.md](file:///home/nguyen-thanh-hung/.gemini/antigravity/skills/grep-app-search/SKILL.md)
- [Script grep_search.py](file:///home/nguyen-thanh-hung/.gemini/antigravity/skills/grep-app-search/scripts/grep_search.py)
