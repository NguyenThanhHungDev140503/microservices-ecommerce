# Active Context

## Trọng tâm hiện tại
- Tiến hành re-index mã nguồn backend sử dụng skill `code-graph-rag-mcp`.

## Các thay đổi gần đây
- Khởi tạo Memory Bank cho dự án.
- Phân tích cấu trúc thư mục backend trong `src/`.

## Các vấn đề đang giải quyết
- Thực hiện index các thư mục: `ApiGateway`, `JobOrchestrator`, `Services`, `Shared`.
- Loại trừ thư mục `Apps` (frontend) khỏi quá trình index.

## Quyết định kỹ thuật
- Sử dụng skill `code-graph-rag-mcp` để tạo clean index cho backend, giúp việc phân tích và tìm kiếm sau này hiệu quả hơn.
