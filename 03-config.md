# Phần 3: Config System — openclaw.json

---

## 3.1 Cấu trúc config đầy đủ có giải thích

```json
{
  "gateway": {
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "{{ KHÔNG hardcode — dùng openclaw config set }}"
    },
    "controlUi": {
      "allowInsecureAuth": false
    }
  },

  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "model": {
        "primary": "ninerouter/stable-stack",
        "fallbacks": [
          "ninerouter/zero-cost"
        ]
      },
      "heartbeat": {
        "enabled": true,
        "every": "30m",
        "model": "ninerouter/zero-cost",
        "target": "last",
        "lightContext": true,
        "isolatedSession": true,
        "activeHours": {
          "start": "08:00",
          "end": "22:00",
          "tz": "Asia/Ho_Chi_Minh"
        },
        "ackMaxChars": 300
      },
      "sandbox": {
        "browser": {
          "allowHostControl": false
        }
      }
    },

    "list": {
      "coder": {
        "workspace": "~/.openclaw/workspaces/coder",
        "model": { "primary": "ninerouter/code-stack" }
      },
      "assistant": {
        "workspace": "~/.openclaw/workspaces/assistant",
        "model": { "primary": "ninerouter/zero-cost" }
      }
    }
  },

  "models": {
    "mode": "merge",
    "providers": {
      "ninerouter": {
        "baseUrl": "http://127.0.0.1:20128/v1",
        "apiKey": "{{ NINEROUTER_API_KEY }}",
        "models": [
          "stable-stack",
          "zero-cost",
          "code-stack"
        ]
      },
      "anthropic": {
        "apiKey": "{{ từ .env: ANTHROPIC_API_KEY }}",
        "models": ["claude-opus-4-6", "claude-sonnet-4-6", "claude-haiku-4-5"]
      },
      "openai": {
        "apiKey": "{{ OPENAI_API_KEY }}",
        "models": ["gpt-5.1-codex", "gpt-4.1", "o3"]
      },
      "openrouter": {
        "baseUrl": "https://openrouter.ai/api/v1",
        "apiKey": "{{ OPENROUTER_API_KEY }}",
        "models": [
          "google/gemini-2.5-pro",
          "google/gemini-2.5-flash",
          "deepseek/deepseek-chat",
          "anthropic/claude-sonnet-4-6"
        ]
      },
      "ollama": {
        "baseUrl": "http://localhost:11434/v1",
        "apiKey": "ollama",
        "models": ["llama3.2:latest", "qwen2.5-coder:7b", "mistral:latest"]
      }
    }
  },

  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "{{ TELEGRAM_BOT_TOKEN từ .env }}",
      "dmPolicy": "allowlist",
      "allowFrom": ["123456789"],
      "groupPolicy": "mention",
      "streamMode": "partial"
    },
    "discord": {
      "enabled": false,
      "token": "{{ DISCORD_BOT_TOKEN }}"
    }
  },

  "tools": {
    "profile": "default",
    "web": {
      "search": {
        "provider": "brave",
        "enabled": true
      },
      "fetch": { "enabled": true }
    }
  },

  "browser": {
    "enabled": true,
    "headless": false,
    "noSandbox": false,
    "defaultProfile": "openclaw",
    "profiles": {
      "openclaw": { "cdpPort": 18800 },
      "work": { "cdpPort": 18801 },
      "shopping": { "cdpPort": 18802 }
    },
    "ssrfPolicy": {
      "dangerouslyAllowPrivateNetwork": false
    }
  },

  "plugins": {
    "entries": {
      "browser": { "enabled": true },
      "ollama": { "enabled": false },
      "memory-core": { "enabled": true },
      "searxng": {
        "enabled": false,
        "config": {
          "webSearch": { "baseUrl": "http://localhost:8889" }
        }
      },
      "telegram": { "enabled": true }
    }
  }
}
```

### Giải thích các field quan trọng

**`gateway.bind`**

| Giá trị | Ý nghĩa | Dùng khi |
|---------|---------|---------|
| `loopback` | Chỉ localhost | Dev/personal trên một máy |
| `tailnet` | Qua Tailscale VPN | Remote access an toàn |
| `any` | Public network | ⚠️ Chỉ khi có firewall + auth token mạnh |

**`agents.defaults.model.fallbacks`**: Danh sách model thử theo thứ tự khi primary bị lỗi/rate-limit. Khi dùng 9Router, fallback thường chỉ cần `ninerouter/zero-cost` vì bản thân 9Router đã tự xử lý fallback giữa các provider bên trong.

**`models.mode`**:
- `merge` — merge providers của bạn với built-in defaults
- `replace` — chỉ dùng providers bạn khai báo

---

## 3.2 CLI config commands

```bash
# Đọc giá trị
openclaw config get agents.defaults.model.primary
openclaw config get channels.telegram.allowFrom

# Ghi giá trị đơn
openclaw config set agents.defaults.model.primary "ninerouter/stable-stack"

# Ghi giá trị mảng (phải là JSON string hợp lệ)
openclaw config set channels.telegram.allowFrom '["123456789", "987654321"]'

# Ghi object lồng nhau
openclaw config set agents.defaults.heartbeat.activeHours '{"start":"08:00","end":"22:00","tz":"Asia/Ho_Chi_Minh"}'

# Xóa key
openclaw config unset models.providers.ollama

# Tiện ích
openclaw config file          # In đường dẫn file config
openclaw config validate      # Validate JSON schema, báo lỗi nếu sai

# Restart sau khi đổi config quan trọng
openclaw gateway restart
```

---

## 3.3 Model providers — cách thêm chi tiết

### 9Router (**khuyến nghị — dùng làm primary**)

9Router là proxy chạy local (port 20128) tự động fallback giữa nhiều model. Thêm vào config sau khi đã [cài và cấu hình 9Router](./09b-9router.md):

```bash
# 1. Lấy API key từ 9Router Dashboard → Settings → API Key
# 2. Thêm vào .env
echo 'NINEROUTER_API_KEY=nr-xxxx' >> ~/.openclaw/.env

# 3. Config provider (dùng config edit vì thêm nguyên block)
openclaw config edit
```

Block cần thêm vào `models.providers`:

```json
"ninerouter": {
  "baseUrl": "http://127.0.0.1:20128/v1",
  "apiKey": "$NINEROUTER_API_KEY",
  "models": ["stable-stack", "zero-cost", "code-stack"]
}
```

Đổi primary sang 9Router:

```bash
openclaw config set agents.defaults.model.primary "ninerouter/stable-stack"
openclaw config set agents.defaults.model.fallbacks '["ninerouter/zero-cost"]'
openclaw config set agents.defaults.heartbeat.model "ninerouter/zero-cost"
openclaw gateway restart
```

> **Lưu ý**: Dùng `127.0.0.1` thay vì `localhost` — OpenClaw có bug IPv6 resolution với `localhost`. Xem chi tiết combo setup tại [Phần 9B](./09b-9router.md).

### Anthropic Claude

```bash
# Thêm API key vào .env
echo 'ANTHROPIC_API_KEY=sk-ant-xxxx' >> ~/.openclaw/.env

# Config model
openclaw config set agents.defaults.model.primary "anthropic/claude-sonnet-4-6"
```

Models có sẵn: `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`

### OpenAI GPT

```bash
echo 'OPENAI_API_KEY=sk-xxxx' >> ~/.openclaw/.env
openclaw config set models.providers.openai.apiKey '$OPENAI_API_KEY'
openclaw config set models.providers.openai.models '["gpt-5.1-codex","gpt-4.1","o3"]'
```

### Google Gemini (trực tiếp)

```bash
openclaw config set models.providers.google.baseUrl "https://generativelanguage.googleapis.com/v1beta/openai"
openclaw config set models.providers.google.apiKey '$GOOGLE_API_KEY'
openclaw config set models.providers.google.models '["gemini-2.5-pro","gemini-2.5-flash"]'
```

### OpenRouter (một key, 200+ models)

```bash
echo 'OPENROUTER_API_KEY=sk-or-xxxx' >> ~/.openclaw/.env
openclaw config set models.providers.openrouter.baseUrl "https://openrouter.ai/api/v1"
openclaw config set models.providers.openrouter.apiKey '$OPENROUTER_API_KEY'

# Format model: openrouter/<author>/<slug>
openclaw config set agents.defaults.model.primary "openrouter/anthropic/claude-sonnet-4-6"

# Auto model (OpenRouter tự chọn rẻ nhất)
openclaw config set agents.defaults.model.primary "openrouter/openrouter/auto"
```

### Ollama (local, miễn phí, offline)

```bash
# Đảm bảo Ollama đang chạy
ollama serve

# Config
openclaw config set models.providers.ollama.baseUrl "http://localhost:11434/v1"
openclaw config set models.providers.ollama.apiKey "ollama"
openclaw config set models.providers.ollama.models '["llama3.2:latest","qwen2.5-coder:7b"]'

# Dùng cho heartbeat (miễn phí)
openclaw config set agents.defaults.heartbeat.model "ollama/llama3.2:latest"
```

> 💡 Dùng context window tối thiểu 64K tokens cho local models trong agentic tasks.

### LiteLLM proxy (self-hosted, wrap nhiều provider)

```bash
# Giả sử LiteLLM đang chạy ở port 4000
openclaw config set models.providers.litellm.baseUrl "http://localhost:4000/v1"
openclaw config set models.providers.litellm.apiKey "sk-local"
openclaw config set models.providers.litellm.models '["gpt-4","claude-3","gemini-pro"]'
```

### 9Router (smart router — xem chi tiết [Phần 9B](./09b-9router.md))

```bash
# Sau khi cài và cấu hình 9Router, đổi primary về combo
openclaw config set agents.defaults.model.primary "ninerouter/stable-stack"
openclaw gateway restart
```

---

## 3.4 Chiến lược model routing theo task

Khi dùng 9Router, gán combo phù hợp cho từng loại task thay vì chỉ định từng model lẻ:

| Task | Combo khuyến nghị | Lý do |
|------|-------------------|-------|
| Heartbeat checks | `ninerouter/zero-cost` | NVIDIA free → Ollama local, không tốn quota paid |
| Cron tasks đơn giản (tóm tắt, nhắc nhở) | `ninerouter/zero-cost` | Đủ chất lượng, miễn phí |
| Cron tasks phức tạp (phân tích, research) | `ninerouter/stable-stack` | Gemini Pro trước, NVIDIA fallback |
| Browser automation (form filling, scraping) | `ninerouter/stable-stack` | Paid model với vision + reasoning |
| Coding (generate, refactor, debug) | `ninerouter/code-stack` | Claude → Kimi K2.5 → Qwen local |
| Chat thông thường | `ninerouter/stable-stack` | Chất lượng cao, tự động fallback |
| Monitor / agent chạy 24/7 | `ninerouter/zero-cost` | Không quota, không chi phí |

### Cấu hình model theo loại task trong config

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "ninerouter/stable-stack",
        "fallbacks": ["ninerouter/zero-cost"]
      },
      "heartbeat": {
        "model": "ninerouter/zero-cost"
      }
    },
    "list": {
      "coder": {
        "model": { "primary": "ninerouter/code-stack" }
      },
      "monitor": {
        "model": { "primary": "ninerouter/zero-cost" }
      }
    }
  }
}
```

---

## 3.5 Quản lý secrets an toàn

```bash
# ~/.openclaw/.env — file này KHÔNG được commit git
ANTHROPIC_API_KEY=sk-ant-xxxx
OPENAI_API_KEY=sk-xxxx
OPENROUTER_API_KEY=sk-or-xxxx
TELEGRAM_BOT_TOKEN=1234567890:AAFxxxx
GOOGLE_API_KEY=AIzaxxxx

# Tham chiếu trong openclaw.json dùng $VAR_NAME syntax
# openclaw tự load .env khi start
```

```bash
# Cách bảo mật nhất — dùng system keychain
openclaw auth set anthropic sk-ant-xxxx
```

> ⚠️ Không bao giờ hardcode API key trực tiếp vào `openclaw.json`. File này thường nằm trong thư mục config có thể vô tình bị share.
