# Phần 9B: 9Router — Fallback Tự Động Giữa Các Model

> **Repository**: [github.com/decolua/9router](https://github.com/decolua/9router)
> **Port**: 20128 | **Dashboard**: `http://localhost:20128`
> **Triết lý**: Paid API → NVIDIA Free → Ollama Local — không bao giờ gián đoạn

---

## Mục tiêu

Dùng 9Router như một **proxy trung gian** giữa OpenClaw và nhiều model providers. Khi model A hết quota hay lỗi, 9Router **tự động chuyển sang model B** mà không làm gián đoạn workflow.

```
OpenClaw
    ↓ gửi request tới 9Router (port 20128)
9Router
    ├── Tier 1: Paid API (Gemini Pro / Claude) — chất lượng cao, trả phí
    │       ↓ hết quota hoặc lỗi
    ├── Tier 2: NVIDIA free API — miễn phí, model mạnh
    │       ↓ hết rate limit
    └── Tier 3: Ollama local — luôn sẵn sàng, zero cost, không quota
```

---

## Bước 1: Cài 9Router

```bash
# Cài vào Node.js hệ thống (global) — KHÔNG dùng npm của openclaw
# 9Router là server độc lập, openclaw kết nối qua HTTP, không import module
npm install -g 9router
9router
# Dashboard tự mở tại http://localhost:20128
```

Hoặc chạy ngầm như service:

```bash
# Chạy background
nohup 9router > ~/.9router.log 2>&1 &

# Kiểm tra đang chạy
curl http://localhost:20128/health
```

---

## Bước 2: Lấy API key của từng provider

### Paid APIs (chọn ít nhất 1)

**Gemini Pro** (Google):
```
https://aistudio.google.com/apikey
→ Create API Key → Copy (format: AIza...)
```

**Claude** (Anthropic):
```
https://console.anthropic.com/settings/keys
→ Create Key → Copy (format: sk-ant-...)
```

**OpenRouter** (optional — 1 key cho 200+ models):
```
https://openrouter.ai/keys
→ Create Key → Copy (format: sk-or-...)
```

### NVIDIA free API

1. Đăng ký tại [build.nvidia.com](https://build.nvidia.com) (miễn phí)
2. Vào Settings → API Keys → Generate Key
3. Copy key (format: `nvapi-...`)
4. **Base URL**: `https://integrate.api.nvidia.com/v1`

> Signup nhận 1,000 credits miễn phí. Một số model hoàn toàn free không giới hạn.

### Ollama (local)

```bash
# Cài Ollama (nếu chưa có)
# macOS
brew install ollama

# Linux
curl -fsSL https://ollama.com/install.sh | sh

# Kéo model về
ollama pull llama3.2          # 2GB — tốt cho chat thường
ollama pull qwen2.5-coder:7b  # 4.7GB — tốt cho code
ollama pull mistral:latest    # 4.1GB — balanced

# Kiểm tra Ollama đang chạy
curl http://localhost:11434/api/tags

# Ollama expose OpenAI-compatible API tại:
# http://localhost:11434/v1
```

---

## Bước 3: Thêm providers vào 9Router Dashboard

Mở `http://localhost:20128` → **Providers** → **Add Provider**

### Thêm Gemini

```
Provider type: Gemini (hoặc "OpenAI compatible")
API Key: AIza...
Base URL: https://generativelanguage.googleapis.com/v1beta/openai
Model: gemini-2.5-pro
```

### Thêm Anthropic (Claude)

```
Provider type: Anthropic
API Key: sk-ant-...
Model: claude-sonnet-4-6 hoặc claude-opus-4-6
```

### Thêm OpenRouter

```
Provider type: OpenRouter (hoặc OpenAI compatible)
API Key: sk-or-...
Base URL: https://openrouter.ai/api/v1
Models: google/gemini-2.5-pro, deepseek/deepseek-chat, ...
```

### Thêm NVIDIA free API

```
Provider type: OpenAI compatible (Custom)
Name: NVIDIA NIM
API Key: nvapi-...
Base URL: https://integrate.api.nvidia.com/v1
```

Models NVIDIA free tốt nhất để dùng:

| Model ID | Kích thước | Tốt cho |
|----------|-----------|---------|
| `nvidia/llama-3.1-nemotron-ultra-253b-v1` | 253B | Reasoning, code phức tạp |
| `nvidia/llama-3.3-nemotron-super-49b-v1` | 49B | Chat, phân tích |
| `nvidia/llama-3.1-nemotron-70b-instruct` | 70B | Đa năng |
| `nvidia/mistral-nemo-minitron-8b-8k-instruct` | 8B | Nhanh, nhẹ |
| `moonshotai/kimi-k2.5` | — | Code, reasoning |
| `minimaxai/minimax-m2.5` | — | Long context |

### Thêm Ollama (local)

```
Provider type: OpenAI compatible (Custom)
Name: Ollama Local
API Key: ollama          ← bất kỳ string nào đều được
Base URL: http://localhost:11434/v1
```

> ⚠️ **WSL2**: Nếu 9Router chạy trong WSL2 và Ollama chạy trên Windows host, dùng `http://host.docker.internal:11434/v1` thay vì `localhost`.

Models Ollama để thêm vào:
```
llama3.2:latest
qwen2.5-coder:7b
mistral:latest
```

---

## Bước 4: Tạo Combo fallback

Dashboard → **Combos** → **New Combo**

### Combo khuyến nghị — "stable-stack"

```
Name: stable-stack

Tier 1: gemini/gemini-2.5-pro          (paid, chất lượng cao nhất)
Tier 2: nvidia/llama-3.1-nemotron-ultra-253b-v1  (free NVIDIA)
Tier 3: ollama/llama3.2:latest          (local, luôn có)
```

### Combo miễn phí hoàn toàn — "zero-cost"

```
Name: zero-cost

Tier 1: nvidia/llama-3.3-nemotron-super-49b-v1   (NVIDIA free)
Tier 2: nvidia/llama-3.1-nemotron-70b-instruct   (NVIDIA free backup)
Tier 3: ollama/qwen2.5-coder:7b                  (local fallback)
```

### Combo cho coding — "code-stack"

```
Name: code-stack

Tier 1: anthropic/claude-sonnet-4-6    (paid, tốt nhất cho code)
Tier 2: moonshotai/kimi-k2.5           (NVIDIA free, giỏi code)
Tier 3: ollama/qwen2.5-coder:7b        (local coder model)
```

### Logic fallback

Khi nào 9Router chuyển tier:
- Model trả về lỗi `429` (rate limit / quota hết)
- Model trả về lỗi `503` (overloaded)
- Authentication failure → auto refresh token
- Timeout vượt ngưỡng
- Kết nối thất bại

---

## Bước 5: Kết nối OpenClaw với 9Router

> ⚠️ **Dùng `127.0.0.1` thay vì `localhost`** — OpenClaw có bug IPv6 resolution với `localhost`.

### Lấy API key từ 9Router

```
Dashboard → Settings (icon góc trên phải) → API Key → Copy
```

### Config openclaw.json

> **Không xoá provider cũ** — nếu đang có sẵn provider khác (NVIDIA, Gemini...), giữ nguyên và chỉ **thêm `ninerouter` vào cạnh**. Provider cũ vẫn dùng được để debug hoặc bypass 9Router khi cần. 9Router tự quản lý các API key của từng provider trong dashboard của nó — không liên quan đến provider trong `openclaw.json`.

Chỉ thêm/sửa hai phần sau vào file hiện có:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "provider-cu": { "...giữ nguyên..." },
      "ninerouter": {
        "baseUrl": "http://127.0.0.1:20128/v1",
        "apiKey": "KEY_LẤY_TỪ_9ROUTER_DASHBOARD",
        "models": [
          "stable-stack",
          "zero-cost",
          "code-stack"
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "ninerouter/stable-stack",
        "fallbacks": ["ninerouter/zero-cost"]
      },
      "heartbeat": {
        "model": "ninerouter/zero-cost"
      }
    }
  }
}
```

### CLI commands

> ⚠️ **Không dùng `config set` từng dòng khi thêm provider mới** — openclaw validate toàn bộ object mỗi lần set, sẽ báo lỗi `models: expected array, received undefined` nếu chưa đủ fields. Dùng `openclaw config edit` để thêm cả block một lúc.

```bash
# Mở editor để thêm toàn bộ block ninerouter một lúc
openclaw config edit

# Sau khi lưu file, restart gateway
openclaw gateway restart

# config set chỉ dùng được khi provider đã tồn tại đầy đủ rồi, ví dụ đổi key:
openclaw config set models.providers.ninerouter.apiKey "NEW_KEY"
openclaw gateway restart
```

### Kiểm tra kết nối

```bash
# Test 9Router trực tiếp
curl http://127.0.0.1:20128/v1/models \
  -H "Authorization: Bearer YOUR_9ROUTER_KEY"

# Test qua OpenClaw
openclaw agent --to "test" --message "Mô hình nào đang trả lời câu này?" --local
```

---

## Bước 6: Chiến lược dùng combo theo task

Thay vì dùng một combo cho mọi thứ, gán từng agent hoặc cron job dùng combo phù hợp:

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "ninerouter/stable-stack" }
    },
    "list": {
      "monitor": {
        "model": { "primary": "ninerouter/zero-cost" }
      },
      "coder": {
        "model": { "primary": "ninerouter/code-stack" }
      }
    }
  }
}
```

**Heartbeat** (check định kỳ) → `zero-cost` — không cần model mạnh, tiết kiệm quota paid

**Cron jobs** (tóm tắt, brief) → `zero-cost` hoặc `stable-stack` tuỳ độ quan trọng

**Chat thường** → `stable-stack` — Gemini trước, NVIDIA fallback nếu hết

**Coding tasks** → `code-stack` — Claude trước, Kimi K2.5, rồi Qwen local

---

## Monitoring trên Dashboard

`http://localhost:20128` hiển thị real-time:

- **Quota còn lại** cho mỗi provider — biết khi nào sắp hết
- **Reset countdown** — biết bao giờ quota refresh (5h rolling / daily / weekly)
- **Request logs** — xem routing thực tế, model nào đang xử lý
- **Cost estimate** — so sánh chi phí (chỉ để tham khảo, không phải bill)

---

## Cài 9Router như system service (tự start khi boot)

### Linux (systemd)

```bash
# Tạo service file
sudo tee /etc/systemd/system/9router.service << 'EOF'
[Unit]
Description=9Router AI Proxy
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/home/$USER
ExecStart=/usr/local/bin/9router
Environment=PORT=20128
Environment=NEXT_PUBLIC_BASE_URL=http://localhost:20128
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable 9router
sudo systemctl start 9router
sudo systemctl status 9router
```

### macOS (launchd)

```bash
cat > ~/Library/LaunchAgents/ai.9router.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.9router</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/9router</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PORT</key>
        <string>20128</string>
        <key>NEXT_PUBLIC_BASE_URL</key>
        <string>http://localhost:20128</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/9router.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/9router.err</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/ai.9router.plist
launchctl list | grep 9router
```

### Docker Compose (khuyến nghị cho VPS)

```yaml
# docker-compose.yml
version: "3"
services:
  9router:
    image: node:24-alpine
    working_dir: /app
    command: sh -c "npm install -g 9router && 9router"
    ports:
      - "127.0.0.1:20128:20128"   # Chỉ bind loopback
    environment:
      PORT: 20128
      HOSTNAME: 0.0.0.0
      NEXT_PUBLIC_BASE_URL: http://localhost:20128
    volumes:
      - ./9router-data:/root/.9router
    restart: unless-stopped
```

```bash
docker compose up -d
docker compose logs -f 9router
```

---

## Environment variables đầy đủ

| Variable | Mặc định | Ý nghĩa |
|----------|---------|---------|
| `PORT` | `20128` | Port server |
| `HOSTNAME` | `localhost` | Bind address — dùng `0.0.0.0` cho VPS |
| `NEXT_PUBLIC_BASE_URL` | `http://localhost:20128` | URL dashboard |
| `DATA_DIR` | `./data` | Thư mục lưu database/config |
| `NODE_ENV` | `production` | Environment |

---

## Troubleshooting

| Vấn đề | Nguyên nhân | Fix |
|--------|-------------|-----|
| OpenClaw không kết nối được | IPv6 resolution | Dùng `127.0.0.1` thay vì `localhost` |
| Ollama không nhận | Ollama chưa start | `ollama serve` hoặc kiểm tra service |
| NVIDIA trả về 401 | API key sai format | Key phải bắt đầu bằng `nvapi-` |
| Combo không fallback | Config combo sai tier | Vào Dashboard → Combo → kiểm tra 3 tiers |
| 9Router crash | Port conflict | `lsof -i :20128`, kill process cũ |
| WSL2: Ollama không thấy | Host networking | Dùng `http://host.docker.internal:11434/v1` |
