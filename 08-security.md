# Phần 8: Security & Production Hardening

---

## 8.1 Threat model

OpenClaw được thiết kế cho **personal use** với trust model: một operator (bạn), một gateway, nhiều trusted channels. Không phải multi-tenant hostile environment.

Rủi ro thực tế cần chú ý:
- **Unauthorized channel access**: Ai đó nhắn tin vào bot của bạn và điều khiển agent
- **API key exposure**: Key bị lộ qua config file, log, hoặc workspace files
- **Skill supply chain**: Skill độc hại inject vào context, leak data hoặc chạy code nguy hiểm
- **Open gateway**: Gateway expose ra internet mà không có auth

---

## 8.2 Security checklist tối thiểu

### Config bắt buộc

```json
{
  "gateway": {
    "auth": {
      "mode": "token"
    },
    "bind": "loopback",
    "controlUi": {
      "allowInsecureAuth": false
    }
  },
  "channels": {
    "telegram": {
      "dmPolicy": "allowlist",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID_ONLY"],
      "groupPolicy": "mention"
    }
  }
}
```

### Thiết lập qua CLI

```bash
# Auth token mạnh
openclaw config set gateway.auth.mode token
openclaw config set gateway.auth.token "$(openssl rand -hex 32)"

# Chỉ bind loopback (không expose ra network)
openclaw config set gateway.bind loopback

# Tắt insecure auth trên Control UI
openclaw config set gateway.controlUi.allowInsecureAuth false

# Telegram allowlist
openclaw config set channels.telegram.dmPolicy allowlist
openclaw config set channels.telegram.allowFrom '["YOUR_USER_ID"]'

# Restart
openclaw gateway restart
```

---

## 8.3 Secrets management

```bash
# API keys trong .env, KHÔNG trong openclaw.json
cat ~/.openclaw/.env
# ANTHROPIC_API_KEY=sk-ant-xxx
# OPENAI_API_KEY=sk-xxx
# TELEGRAM_BOT_TOKEN=xxx

# Tham chiếu trong config dùng $VAR_NAME
openclaw config set models.providers.anthropic.apiKey '$ANTHROPIC_API_KEY'

# Cách bảo mật nhất — system keychain
openclaw auth set anthropic sk-ant-xxx
openclaw auth set telegram-bot xxx

# Xem đã lưu keys nào
openclaw auth list
```

### .gitignore nếu backup config

```gitignore
# ~/.openclaw/ trong git repo
.env
blockrun/wallet.key
cron/jobs-state.json
devices/paired.json
devices/pending.json
workspace/USER.md
workspace/MEMORY.md
workspace/memory/
```

> ⚠️ Đừng bao giờ commit `.env` hoặc `wallet.key` lên git — kể cả private repo.

---

## 8.4 Security audit

```bash
# Audit tự động
openclaw security audit

# Audit sâu hơn
openclaw security audit --deep

# Auto-fix common issues (khuyến nghị chạy sau cài đặt)
openclaw security audit --fix
```

`--fix` tự động:
- Chuyển group policy về `mention` nếu đang `open`
- Enable `logging.redactSensitive: "tools"` để không log credentials
- Tighten filesystem permissions
- Cảnh báo về skills có permission cao nguy hiểm

---

## 8.5 Skill security

```bash
# Trước khi cài bất kỳ skill nào
openclaw skills info <skill-slug>

# Kiểm tra permissions — cờ đỏ:
# shell.execute + fs.read_root  → CÓ THỂ ĐỌC TOÀN BỘ FS VÀ CHẠY SHELL
# network.unrestricted           → GỌI BẤT KỲ URL NÀO

# Chạy skill auditor (nếu đã cài)
openclaw skills install skill-security-auditor
# Sau đó:
# → "Audit skill agent-browser trước khi cài"
```

---

## 8.6 Remote access an toàn — Tailscale

Nếu cần truy cập Gateway từ xa (không phải localhost), dùng Tailscale thay vì expose public:

```bash
# Cài Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Config gateway bind tailnet
openclaw config set gateway.bind tailnet
openclaw gateway restart

# Truy cập từ device khác trong tailnet
# Gateway URL: http://<tailscale-machine-name>:18789
```

---

## 8.7 Backup & Recovery

```bash
# Tạo backup
openclaw backup create
# → Tạo file: ~/.openclaw/backups/openclaw-2026-04-23-143022.tar.gz

# Verify backup
openclaw backup verify ~/.openclaw/backups/openclaw-2026-04-23-143022.tar.gz

# Restore (trên máy mới hoặc sau khi mất data)
openclaw backup restore ~/.openclaw/backups/openclaw-2026-04-23-143022.tar.gz

# Liệt kê backups
openclaw backup list
```

### Cron backup tự động

```bash
openclaw cron add \
  --name "weekly-backup" \
  --cron "0 2 * * 0" \
  --tz "Asia/Ho_Chi_Minh" \
  --session isolated \
  --announce \
  --channel telegram \
  --message "Chạy: openclaw backup create. Xác nhận thành công và log đường dẫn file backup."
```

---

## 8.8 Update

```bash
# Kiểm tra version mới
openclaw update --check
# → Current: 2026.4.21, Latest: 2026.4.21 (up to date)

# Update
openclaw update

# Hoặc qua npm
npm update -g openclaw

# Verify sau update
openclaw --version
openclaw gateway status
openclaw doctor
```

---

## 8.9 Sandbox mode cho agent

Nếu muốn giới hạn agent không thể làm gì ngoài danh sách tool được phép:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "enabled": true,
        "allowedTools": ["browser", "web_search", "web_fetch"],
        "browser": {
          "allowHostControl": false,
          "allowedProfiles": ["openclaw"]
        }
      }
    }
  }
}
```

---

## 8.10 Logging và monitoring

```bash
# Xem logs gateway
openclaw logs

# Follow logs real-time
openclaw logs --follow

# Logs với level filter
openclaw logs --level error
openclaw logs --level warn

# Logs của specific session
openclaw logs --session <session-id>

# Redact sensitive info trong logs (khuyến nghị production)
openclaw config set logging.redactSensitive "tools"
```

### systemd log (Linux)

```bash
journalctl --user -u openclaw-gateway -f
journalctl --user -u openclaw-gateway --since "1 hour ago"
```
