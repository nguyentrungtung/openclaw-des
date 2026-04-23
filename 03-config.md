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
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": [
          "openai/gpt-4.1",
          "openrouter/google/gemini-2.5-pro"
        ]
      },
      "heartbeat": {
        "enabled": true,
        "every": "30m",
        "model": "openrouter/google/gemini-2.5-flash",
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
        "model": { "primary": "anthropic/claude-opus-4-6" }
      },
      "assistant": {
        "workspace": "~/.openclaw/workspaces/assistant",
        "model": { "primary": "openrouter/google/gemini-2.5-flash" }
      }
    }
  },

  "models": {
    "mode": "merge",
    "providers": {
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

**`agents.defaults.model.fallbacks`**: Danh sách model thử theo thứ tự khi primary bị lỗi/rate-limit. Không cần ClawRouter để có basic fallback.

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
openclaw config set agents.defaults.model.primary "anthropic/claude-sonnet-4-6"

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

### ClawRouter (smart router — xem chi tiết Phần 9)

```bash
# Sau khi cài ClawRouter
openclaw config set agents.defaults.model.primary "blockrun/auto"
```

---

## 3.4 Chiến lược model routing theo task

| Task | Model khuyến nghị | Lý do |
|------|-------------------|-------|
| Heartbeat checks | `ollama/llama3.2` hoặc `gemini-2.5-flash` | Rẻ/miễn phí, đủ cho monitoring |
| Cron tasks đơn giản (tóm tắt, nhắc nhở) | `deepseek/deepseek-chat` | Cost-effective, text quality tốt |
| Cron tasks phức tạp (phân tích, research) | `gemini-2.5-pro` | Mạnh, giá hợp lý |
| Browser automation (form filling, scraping) | `claude-sonnet-4-6` hoặc `gemini-2.5-pro` | Vision + reasoning tốt |
| Coding (generate, refactor, debug) | `claude-opus-4-6` hoặc `gpt-5.1-codex` | Mạnh nhất cho code |
| Chat thông thường | `claude-sonnet-4-6` | Cân bằng chất lượng/giá |
| Toàn bộ tự động | `blockrun/auto` (ClawRouter) | Tự chọn model theo task |

### Cấu hình model theo loại task trong config

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["openrouter/google/gemini-2.5-pro"]
      },
      "heartbeat": {
        "model": "openrouter/google/gemini-2.5-flash"
      }
    },
    "list": {
      "coder": {
        "model": { "primary": "anthropic/claude-opus-4-6" }
      },
      "monitor": {
        "model": { "primary": "ollama/llama3.2:latest" }
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
