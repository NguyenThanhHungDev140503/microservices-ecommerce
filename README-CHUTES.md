# LightRAG with Chutes AI Configuration

Cấu hình LightRAG sử dụng Chutes AI làm LLM và Embedding provider.

## 📁 Cấu trúc thư mục

```
~/LightRAG/
├── .env                    # Environment configuration
├── docker-compose.yml      # Docker Compose setup
├── inputs/                 # Directory for input documents
├── rag_storage/            # Storage for RAG data
└── README-CHUTES.md        # This file
```

## 🔧 Cấu hình

### 1. Lấy Chutes API Key

1. Truy cập: https://chutes.ai/
2. Đăng nhập / Tạo tài khoản
3. Vào **Settings** → **API Keys**
4. Tạo key mới và copy

### 2. Set API Key

**Cách 1: Export environment variable (khuyến nghị)**
```bash
export CHUTES_API_KEY=cp-your-api-key-here
```

**Cách 2: Thêm vào ~/.bashrc hoặc ~/.zshrc**
```bash
echo 'export CHUTES_API_KEY=cp-your-api-key-here' >> ~/.bashrc
source ~/.bashrc
```

## 🚀 Chạy LightRAG

### Cách 1: Sử dụng Docker (Khuyến nghị)

```bash
# Chạy với Docker Compose
docker-compose up -d

# Kiểm tra logs
docker-compose logs -f

# Dừng service
docker-compose down
```

### Cách 2: Chạy trực tiếp với Python

```bash
# Cài đặt dependencies
pip install -e .

# Chạy server
lightrag-server
```

## 📝 Sử dụng

### 1. Upload Documents

```bash
# Copy files vào inputs directory
cp /path/to/your/documents/* ./inputs/

# Hoặc dùng API
curl -X POST "http://localhost:9621/documents/batch" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@document.pdf"
```

### 2. Query

```bash
# Local mode - specific entities
curl -X POST "http://localhost:9621/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How does authentication work?",
    "mode": "local"
  }'

# Global mode - big picture
curl -X POST "http://localhost:9621/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the system architecture?",
    "mode": "global"
  }'

# Hybrid mode - best of both
curl -X POST "http://localhost:9621/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Find all usages of UserService",
    "mode": "hybrid"
  }'
```

### 3. WebUI

Truy cập: http://localhost:9621/webui

## ⚙️ Configuration Details

### LLM Configuration (Chutes)
- **Provider**: Chutes AI
- **Model**: `chutesai/llama-4-scout` (128K context)
- **API**: https://api.chutes.ai/v1
- **Max Tokens**: 9000

### Embedding Configuration (Chutes)
- **Provider**: Chutes AI
- **Model**: `chutesai/nomic-embed-text`
- **Dimensions**: 768
- **API**: https://api.chutes.ai/v1

### Available Chutes Models

| Model | Context | Use Case |
|-------|---------|----------|
| `chutesai/llama-4-scout` | 128K | General purpose, fast |
| `chutesai/llama-4-maverick` | 128K | Complex reasoning |
| `chutesai/deepseek-v3` | 64K | Code, analysis |
| `chutesai/qwen-2.5-72b` | 32K | Multilingual |
| `chutesai/nomic-embed-text` | - | Embeddings |

Để đổi model, edit `.env` và restart:
```bash
LLM_MODEL=chutesai/llama-4-maverick
```

## 🔍 Query Modes

| Mode | Description | Best For |
|------|-------------|----------|
| `naive` | Vector similarity only | Simple Q&A |
| `local` | Entity-centric retrieval | Specific entities |
| `global` | Relationship-focused | Big picture questions |
| `hybrid` | Combined approach | Complex queries |
| `mix` | Merge KG + vector results | Balanced results |
| `bypass` | Direct LLM | Quick answers |

## 🛠️ Troubleshooting

### Issue: "Invalid API Key"
```bash
# Kiểm tra API key đã set chưa
echo $CHUTES_API_KEY

# Nếu chưa, set lại
export CHUTES_API_KEY=cp-your-api-key-here
```

### Issue: "Connection refused"
```bash
# Kiểm tra container đang chạy
docker ps

# Restart nếu cần
docker-compose restart
```

### Issue: "Rate limit"
- Chutes có rate limits cho free tier
- Giảm `MAX_ASYNC` và `MAX_PARALLEL_INSERT` trong `.env`
- Hoặc nâng cấp plan

## 📊 Monitoring

```bash
# Health check
curl http://localhost:9621/health

# Pipeline status
curl http://localhost:9621/documents/pipeline_status

# Graph stats
curl "http://localhost:9621/graph?label=list"
```

## 🔗 Integration với OpenCode

Thêm vào OpenCode MCP configuration:

```json
{
  "mcpServers": {
    "lightrag": {
      "url": "http://localhost:9621",
      "headers": {
        "X-API-Key": "your-lightrag-api-key"
      }
    }
  }
}
```

## 📝 Notes

- Dữ liệu được lưu trong `./rag_storage/` (persisted qua Docker volumes)
- LLM cache được enable để tiết kiệm tokens
- Sử dụng JSON storage (file-based) cho testing
- Đổi sang PostgreSQL cho production

## 📚 Resources

- [LightRAG GitHub](https://github.com/HKUDS/LightRAG)
- [Chutes AI](https://chutes.ai/)
- [Chutes API Docs](https://docs.chutes.ai/)
