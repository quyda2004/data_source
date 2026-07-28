# Badminton — MCP Server (FastMCP wrap 6 tool)

## MCP Server — tái sử dụng bộ tool

**Source:** https://github.com/quyda2004/AI_badminton/blob/main/ai_service/mcp_server.py

Cùng bộ 6 tool đã dùng trong ai-service được wrap thành **MCP Server** (`FastMCP`) để
Claude Desktop hoặc agent khác dùng chung, chạy độc lập bằng:

```bash
python -m ai_service.mcp_server
```

MCP (Model Context Protocol) là chuẩn của Anthropic để agent client (như Claude
Desktop) kết nối tới các "tool provider" bên ngoài — nhờ vậy user Claude Desktop có
thể đặt sân qua cùng bộ tool mà không cần bật full stack `nginx + backend + ai +
postgres + mongo`.

**Bộ 6 tool wrap:**
- `get_courts`
- `check_availability`
- `create_booking`
- `get_my_bookings`
- `cancel_booking`
- `cancel_booking_by_info`

> **Trạng thái thật:** các tool trong MCP dùng token rỗng (`_TOKEN = ""`). Nên tool
> cần auth (`create_booking`, `get_my_bookings`, `cancel_booking`...) sẽ **fail khi
> chạy qua MCP** vì thiếu token hợp lệ — trừ tool nhận `token` trực tiếp làm tham số.
> Comment trong file cũng ghi chú điều này. Đây là scaffolding để tích hợp thật sau
> (truyền token qua env/config), chưa hoàn chỉnh.

**Cách hoàn thiện production (chưa làm):**
- Truyền `SECRET_KEY` + user credential qua env var → MCP server tự đăng nhập lấy JWT
  → set vào `_TOKEN`.
- Hoặc: expose thêm 1 tool `login(email, password)` trong MCP → user Claude Desktop
  đăng nhập 1 lần → subsequent tool call dùng token đó.
- Hoặc: dùng cơ chế OAuth flow của MCP nếu Claude Desktop hỗ trợ.
