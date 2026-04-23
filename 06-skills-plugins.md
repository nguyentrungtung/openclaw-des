# Phần 6: Skills & Plugins

---

## 6.1 Phân biệt Skill vs Plugin

| | Skill | Plugin |
|--|-------|--------|
| **Là gì** | File `SKILL.md` + optional scripts | Module Node.js |
| **Cài từ** | ClawHub registry (`clawhub.ai`) | npm hoặc bundled sẵn |
| **Tác dụng** | Inject vào context agent — dạy agent làm gì bằng ngôn ngữ tự nhiên | Chạy ở tầng Gateway infrastructure, thêm tool/capability |
| **Ví dụ** | `agent-browser` dạy browser workflow | Plugin `browser` cung cấp CDP tool thực sự |
| **Scope** | Agent-level (nhân cách, kiến thức) | Gateway-level (tool, channel, provider) |

**Quy tắc ngón tay cái**: Skills dạy agent *biết làm gì*. Plugins cho agent *làm được điều đó*.

Ví dụ: Skill `agent-browser` dạy agent cú pháp và workflow điều khiển browser. Plugin `browser` là engine CDP thực sự thực thi lệnh. Thiếu plugin → skill không hoạt động.

---

## 6.2 Skills management

### Tìm kiếm và cài

```bash
# Tìm kiếm trên ClawHub
openclaw skills search "browser automation"
openclaw skills search "google workspace"
openclaw skills search "pdf"
openclaw skills search "stock"
openclaw skills search "email"

# Xem chi tiết skill trước khi cài
openclaw skills info agent-browser
# → Hiện: mô tả, permissions cần, số stars, VirusTotal report

# Cài đặt
openclaw skills install agent-browser
openclaw skills install gog
openclaw skills install github
```

### Quản lý skills đã cài

```bash
openclaw skills list                      # Danh sách đã cài + trạng thái
openclaw skills info <skill-slug>         # Chi tiết + permissions
openclaw skills update --all              # Update tất cả skills
openclaw skills update agent-browser      # Update 1 skill cụ thể
openclaw skills disable agent-browser     # Tắt (giữ lại)
openclaw skills enable agent-browser      # Bật lại
openclaw skills uninstall <skill-slug>    # Gỡ cài đặt
```

### Load priority

```
Workspace skills (~/.openclaw/workspace/skills/)  ← Ưu tiên cao nhất
    ↓
~/.openclaw/skills/                              ← User-installed
    ↓
Bundled skills (trong package openclaw)          ← Mặc định
```

---

## 6.3 Top skills quan trọng

| Skill | Mục đích | Stars | Cài lệnh |
|-------|----------|-------|---------|
| `agent-browser` | Browser automation — điều hướng, form fill, scraping | ★★★★ | `openclaw skills install agent-browser` |
| `gog` | Google Workspace: Gmail, Calendar, Drive, Docs, Sheets | ★★★★ | `openclaw skills install gog` |
| `github` | GitHub: PR, issues, actions, review, merge | ★★★ | `openclaw skills install github` |
| `local-web-search-skill` | Web search không cần API key | ★★★ | `openclaw skills install local-web-search-skill` |
| `summarize` | Tóm tắt văn bản, trang web, documents | ★★★ | `openclaw skills install summarize` |
| `nano-pdf` | Đọc, trích xuất, phân tích file PDF | ★★★ | `openclaw skills install nano-pdf` |
| `capability-evolver` | Agent tự phân tích và đề xuất cải thiện workflow | ★★ | `openclaw skills install capability-evolver` |

---

## 6.4 Security checklist trước khi cài skill

```bash
# Bước 1: Xem info và permissions
openclaw skills info <skill-slug>
```

Permissions cần xem xét:

| Permission | Mức rủi ro | Ghi chú |
|-----------|-----------|---------|
| `fs.read` (specific paths) | Thấp | Bình thường |
| `fs.read_root` + `shell.execute` | ⚠️ Cao | Có thể đọc toàn bộ filesystem + chạy lệnh shell |
| `network.domains` (specific) | Thấp | Chỉ gọi domain cụ thể |
| `network.unrestricted` | ⚠️ Trung bình | Gọi bất kỳ URL nào |
| `exec.unrestricted` | ⚠️ Rất cao | Chạy lệnh shell không giới hạn |

```bash
# Bước 2: Kiểm tra trên clawhub.ai
# → Xem VirusTotal scan report
# → Xem source code (nếu open source)
# → Đọc reviews từ community

# Bước 3: Cài trong môi trường isolated trước
# → Tạo agent test với sandbox mode, thử skill
# → Nếu OK mới cài vào agent production
```

> ⚠️ Không cài skill từ nguồn không rõ ràng. Skill có quyền truy cập context của agent và có thể leak thông tin.

---

## 6.5 Tạo custom skill

Cấu trúc tối giản của một skill:

```
my-skill/
├── SKILL.md          # Bắt buộc — định nghĩa skill
└── scripts/          # Optional — script helpers
    └── fetch-data.sh
```

**SKILL.md format:**

```markdown
---
name: My Custom Skill
slug: my-skill
version: 1.0.0
description: Skill làm gì đó hữu ích
permissions:
  network:
    domains: ["api.example.com"]
  fs:
    read: ["~/.myapp/config.json"]
---

## Hướng dẫn sử dụng My Skill

Khi user yêu cầu [làm gì đó], hãy:
1. Fetch data từ https://api.example.com/data
2. Parse JSON response
3. Format và trình bày cho user

## Ví dụ

User: "Lấy thông tin XYZ"
Agent: [Gọi API, parse, format, trả về]

## Lưu ý
- Luôn kiểm tra status code trước khi parse
- Cache kết quả nếu API có rate limit
```

---

## 6.6 Plugins management

### List và enable

```bash
# Xem tất cả plugins
openclaw plugins list

# Enable plugin
openclaw config set plugins.entries.browser.enabled true
openclaw config set plugins.entries.ollama.enabled true
openclaw config set plugins.entries.memory-core.enabled true
openclaw config set plugins.entries.searxng.enabled true

# Disable plugin
openclaw config set plugins.entries.ollama.enabled false

# Restart để apply
openclaw gateway restart
```

### Plugin config chi tiết

```bash
# Searxng (self-hosted web search)
openclaw config set plugins.entries.searxng.enabled true
openclaw config set plugins.entries.searxng.config.webSearch.baseUrl "http://localhost:8889"

# Ollama config
openclaw config set plugins.entries.ollama.enabled true
# Ollama tự detect ở localhost:11434

# Memory-core (dreaming + long-term memory)
openclaw config set plugins.entries.memory-core.enabled true
```

### Built-in plugins quan trọng

| Plugin | Chức năng | Config key |
|--------|-----------|------------|
| `browser` | Browser control qua CDP + Playwright | `plugins.entries.browser` |
| `ollama` | Local LLM inference không cần API key | `plugins.entries.ollama` |
| `memory-core` | Long-term memory + dreaming consolidation | `plugins.entries.memory-core` |
| `searxng` | Self-hosted web search (không cần Brave API) | `plugins.entries.searxng` |
| `telegram` | Telegram channel integration | `channels.telegram` |

### Memory-core và Dreaming

Plugin `memory-core` bật tính năng **dreaming**: sau mỗi session, agent tự "ngủ" và consolidate working memory vào long-term MEMORY.md. Tương tự con người xử lý ký ức khi ngủ.

```bash
openclaw config set plugins.entries.memory-core.enabled true

# Dreaming chạy tự động sau session kết thúc
# Có thể trigger thủ công:
openclaw agent --to <session> --message "Consolidate memories từ hôm nay vào MEMORY.md"
```

---

## 6.7 MCP — Model Context Protocol integration

MCP cho phép wrap bất kỳ external tool nào thành server mà agent có thể gọi.

```json
{
  "tools": {
    "mcp": {
      "servers": {
        "obsidian": {
          "command": "npx",
          "args": ["-y", "@anthropic/mcp-server-obsidian"],
          "env": {
            "OBSIDIAN_VAULT": "~/Documents/Obsidian/MyVault"
          }
        },
        "postgres": {
          "command": "npx",
          "args": ["-y", "@anthropic/mcp-server-postgres"],
          "env": {
            "DATABASE_URL": "postgresql://localhost/mydb"
          }
        }
      }
    }
  }
}
```

Sau khi config, agent tự động có thể gọi các tools từ MCP servers như tools native.

### MCP servers phổ biến

| MCP Server | Mục đích |
|-----------|---------|
| `@anthropic/mcp-server-obsidian` | Đọc/ghi Obsidian vault |
| `@anthropic/mcp-server-postgres` | Query PostgreSQL |
| `@anthropic/mcp-server-filesystem` | Extended file operations |
| `@anthropic/mcp-server-brave-search` | Brave Search API |
| `@anthropic/mcp-server-github` | GitHub API |
| `@anthropic/mcp-server-google-maps` | Google Maps |
