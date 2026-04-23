# Phần 1: Kiến Trúc & Định Nghĩa Cốt Lõi

---

## 1.1 Kiến trúc tổng quan

OpenClaw là một **self-hosted agent gateway** — bạn chạy nó trên máy của mình, nó kết nối messaging apps (Telegram, Discord, WhatsApp…) với LLM và các công cụ tự động hóa.

### Skeleton — toàn bộ hạ tầng trên 1 máy

```
YOUR MACHINE
══════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────┐
  │               INTERNET / EXTERNAL SERVICES              │
  │  Telegram API · Discord API · WhatsApp · Gmail · ...    │
  │  Anthropic API · Google Gemini API · OpenRouter · ...   │
  │  NVIDIA NIM API (integrate.api.nvidia.com)              │
  └───────┬─────────────────────────────────┬───────────────┘
          │ inbound messages                │ LLM API calls
          │                                 │
══════════╪═════════════════════════════════╪════════════════
  LOCAL   │                                 │
          ▼                                 │
  ┌───────────────────────────────┐         │
  │   OPENCLAW GATEWAY            │         │
  │   port 18789                  │         │
  │                               │         │
  │  ┌─────────────────────────┐  │         │
  │  │     AGENT RUNTIME       │  │         │
  │  │  SOUL.md  AGENTS.md     │  │         │
  │  │  USER.md  MEMORY.md     │  │         │
  │  │  Heartbeat  Cron        │  │         │
  │  └──────────┬──────────────┘  │         │
  │             │ LLM request     │         │
  │             ▼                 │         │
  │  ┌──────────────────────────┐ │         │
  │  │  9ROUTER (port 20128)   │─┼─────────┘
  │  │                          │ │
  │  │  Combo "stable-stack":   │ │  Tier 1 ──► Gemini Pro API  ✓
  │  │    1. Gemini Pro         │─┼──────────► (quota OK → dùng)
  │  │    2. NVIDIA Nemotron    │ │
  │  │    3. Ollama local       │ │  Tier 2 ──► NVIDIA NIM free ✓
  │  │                          │─┼──────────► (khi Tier 1 hết quota)
  │  │  [auto fallback]         │ │
  │  └──────────────────────────┘ │  Tier 3 ──► Ollama :11434  ✓
  │                               │──────────► (luôn có, zero cost)
  │  ┌─────────────────────────┐  │
  │  │     TOOL LAYER          │  │
  │  │  browser (CDP :18791)   │  │
  │  │  browser profiles       │  │
  │  │  :18800 · :18801 · ...  │  │
  │  │  exec · read/write      │  │
  │  │  web_search · MCP       │  │
  │  └─────────────────────────┘  │
  └───────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │   OLLAMA  (port 11434)                                  │
  │   llama3.2 · qwen2.5-coder · mistral · ...              │
  │   → chạy hoàn toàn offline, không quota, không phí     │
  └─────────────────────────────────────────────────────────┘

══════════════════════════════════════════════════════════════════
```

### Vị trí của 9Router trong stack

9Router **không phải** một phần của OpenClaw. Nó là một **proxy độc lập** đứng giữa OpenClaw và các LLM providers:

```
OpenClaw Agent
      │
      │  gửi request tới model "ninerouter/stable-stack"
      ▼
  9Router :20128          ← OpenClaw chỉ biết đến đây
      │
      ├── Thử Tier 1: Gemini Pro  ──► API Google (internet)
      │         └─ 429 quota? ─────────────────────┐
      │                                             │ fail
      ├── Thử Tier 2: NVIDIA Nemotron ──► NIM API  │
      │         └─ rate limit? ────────────────────┐│
      │                                            ││ fail
      └── Thử Tier 3: Ollama  ──► localhost:11434 ◄┘│
                └─ luôn thành công ◄─────────────────┘
      │
      │  trả response về
      ▼
OpenClaw Agent  (không biết tier nào đã xử lý)
```

OpenClaw chỉ cần biết một địa chỉ duy nhất: `http://127.0.0.1:20128/v1`. Toàn bộ logic fallback xảy ra bên trong 9Router.

---

### Luồng message đầy đủ: Telegram → Response

```
[1] User gửi message trên Telegram
        │
        ▼
[2] Telegram Bot API → webhook → OpenClaw Gateway (:18789)
        │
        ├─ Xác thực: allowList, auth token
        ├─ Tìm session key (từ chat_id/user_id)
        └─ Load workspace: SOUL.md, AGENTS.md, MEMORY.md...
        │
        ▼
[3] Agent Runtime — build prompt (context + message + tools)
        │
        ▼
[4] Gửi tới 9Router (:20128) — model: "stable-stack"
        │
        ├─ [4a] 9Router thử Gemini Pro → Google API
        │         OK? → trả response ──────────────────┐
        │         Fail (429)? → thử tiếp               │
        │                                               │
        ├─ [4b] 9Router thử NVIDIA Nemotron → NIM API  │
        │         OK? → trả response ──────────────────┤
        │         Fail? → thử tiếp                     │
        │                                               │
        └─ [4c] 9Router thử Ollama → localhost:11434   │
                  OK? → trả response ──────────────────┘
        │
        ▼
[5] LLM response về Agent Runtime
        │
        ├─ Cần tool? → gọi browser / exec / web_search
        │                   └─ Tool chạy → trả kết quả
        │                         └─ Lặp lại [4] nếu cần
        │
        └─ Hoàn chỉnh → Gateway gửi reply về Telegram
        │
        ▼
[6] User nhận reply trên Telegram
```

---

### Port map — tất cả services trên máy

```
:11434   Ollama            (local LLM inference)
:18789   OpenClaw Gateway  (agent entrypoint)
:18791   Browser Control   (CDP service)
:18792   Extension Relay   (existing-session Chrome)
:18800   Browser Profile 1 (managed Chromium)
:18801   Browser Profile 2 (managed Chromium)
:20128   9Router Dashboard (model proxy + fallback)
```

### Phân biệt Gateway vs Node Host vs Agent

| Thành phần | Là gì | Chạy ở đâu |
|------------|-------|------------|
| **Gateway** | Tiến trình Node.js trung tâm, điều phối toàn bộ | Máy chủ/laptop của bạn |
| **Agent** | "Nhân cách" AI được định nghĩa bởi workspace files | Trong Gateway process |
| **Node Host** | Thiết bị pair (iOS/Android/laptop khác) mở rộng capabilities | Thiết bị di động |

---

## 1.2 Bảng định nghĩa thuật ngữ

| Thuật ngữ | Định nghĩa chính xác | Ví dụ thực tế |
|-----------|---------------------|---------------|
| **Gateway** | Tiến trình Node.js chạy local, là trung tâm điều phối session, auth, routing giữa channels và agents. Luôn lắng nghe ở port 18789 (configurable). | `openclaw gateway start` — khởi động như một system service |
| **Agent** | Một "nhân cách" AI được định nghĩa bởi workspace files. Một Gateway có thể host nhiều agent song song. | Agent "coder" dùng Claude Opus cho coding, agent "assistant" dùng Gemini Flash cho chat thường |
| **Skill** | Gói `SKILL.md` + optional scripts, dạy agent cách làm việc gì đó bằng ngôn ngữ tự nhiên. Cài từ ClawHub registry. | `agent-browser` dạy agent cách điều khiển browser; `gog` dạy dùng Google Workspace |
| **Plugin** | Module Node.js mở rộng khả năng của Gateway (thêm tool, thêm channel, thêm provider). Bundled hoặc cài qua npm. | Plugin `browser` cung cấp CDP tool; plugin `ollama` kết nối local LLM |
| **Channel** | Giao diện messaging mà Gateway kết nối: Telegram, Discord, WhatsApp, Signal, iMessage, Slack... | Cùng một agent trả lời qua cả Telegram lẫn Discord |
| **Heartbeat** | Cơ chế agent tự thức dậy định kỳ để kiểm tra HEARTBEAT.md và gửi alert nếu cần. Không tạo background task records. | Mỗi 30 phút agent check email quan trọng, chỉ ping bạn nếu có gì cần xem |
| **Cron** | Job scheduler chính xác theo giờ/ngày, chạy trong isolated session, có lịch sử run. | "Mỗi sáng 7h gửi briefing ngày", "Thứ 2 hàng tuần backup workspace" |
| **Hook** | Automation phản ứng với event (session mới, reset, boot, tool call...) không phải theo lịch. | Auto-load project context khi `/new` session |
| **Standing Order** | Quy tắc vận hành viết trong AGENTS.md, agent tuân theo mọi session. | "Luôn xác nhận trước khi xóa file", "Không commit code vào weekend" |
| **Node Host** | Thiết bị di động (iOS/Android) hoặc máy tính được pair với Gateway, có thể extend capabilities (camera, voice, location). | iPhone pair → agent có thể nhận voice command, gửi push notification |
| **Session** | Chuỗi conversation liên tục giữa user và agent, có state và context riêng. Key thường derived từ channel + user ID. | Telegram chat với bạn = 1 session; isolated cron job = 1 session riêng |
| **Workspace** | Thư mục chứa các markdown files định nghĩa agent: SOUL.md, AGENTS.md, MEMORY.md, v.v. | `~/.openclaw/workspace/` |
| **MCP** | Model Context Protocol — chuẩn mở để wrap external tools thành server agent có thể gọi. | Wrap Obsidian vault thành MCP server → agent đọc/ghi notes của bạn |

---

## 1.3 Workspace Files — "Bộ não" của Agent

Thứ tự load precedence: **workspace/** (user-defined) > **managed** (skill-injected) > **bundled** (defaults)

| File | Mục đích | Ai viết | Tần suất đọc |
|------|----------|---------|-------------|
| `SOUL.md` | Nhân cách, tone, giới hạn hành vi, role identity | Bạn viết một lần | Mọi request |
| `AGENTS.md` | Quy trình vận hành, numbered procedures, standing orders | Bạn viết, có thể grow | Mọi request |
| `USER.md` | Profile của bạn: tên, timezone, expertise, preferences | Bạn viết, agent cũng update | Mọi request |
| `MEMORY.md` | Long-term memory cross-session, patterns, facts | Agent tự ghi; bạn seed ban đầu | Mọi request |
| `HEARTBEAT.md` | Checklist tác vụ định kỳ | Bạn viết | Mỗi heartbeat tick |
| `IDENTITY.md` | Display name, agent ID, routing metadata | Bạn viết (ngắn) | Khi routing |
| `TOOLS.md` | Inventory tool có sẵn, usage notes, restrictions | Bạn viết (optional) | Mọi request |
| `memory/YYYY-MM-DD.md` | Working notes hàng ngày | Agent tự ghi | Khi cần context hôm nay |

> **Nguyên tắc phân chia**: Personality → `SOUL.md`. Procedures → `AGENTS.md`. Đừng trộn lẫn — đây là lỗi phổ biến nhất khiến agent hành xử không nhất quán.

---

## 1.4 Multi-agent routing

Một Gateway có thể chạy nhiều agent song song, mỗi agent có workspace riêng:

```json
{
  "agents": {
    "list": {
      "coder": {
        "workspace": "~/.openclaw/workspaces/coder",
        "model": { "primary": "anthropic/claude-opus-4-6" }
      },
      "assistant": {
        "workspace": "~/.openclaw/workspaces/assistant",
        "model": { "primary": "openrouter/google/gemini-2.5-flash" }
      }
    }
  }
}
```

Routing dựa trên channel, session key, hoặc explicit `--agent` flag trong lệnh gọi.
