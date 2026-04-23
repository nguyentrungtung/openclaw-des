# Phần 2: Cài Đặt & Thiết Lập Ban Đầu

---

## 2.1 Prerequisites

| Dependency | Version | Bắt buộc? | Ghi chú |
|------------|---------|-----------|---------|
| Node.js | **24.x** (khuyến nghị) hoặc 22.16+ LTS | Bắt buộc | Node 20 không được hỗ trợ từ v2026.1+ |
| npm / pnpm | Bất kỳ version gần đây | npm đủ dùng | pnpm nhanh hơn cho install |
| Playwright | Cài riêng sau (xem Phần 5) | Cần cho browser snapshot/PDF | Không bundle cùng openclaw |
| Chromium | Tự download qua Playwright | Cho browser automation | Chỉ cần nếu dùng browser tool |
| USDC wallet | Tự tạo qua ClawRouter | Chỉ nếu dùng paid models qua ClawRouter | Miễn phí nếu chỉ dùng free models |

**OS support matrix:**

| OS | Trạng thái | Lưu ý |
|----|-----------|-------|
| macOS 13+ | Đầy đủ | Voice, menu bar, Canvas, iOS node |
| Linux (Ubuntu/Debian/Arch) | Đầy đủ server | Headless browser, daemon, tất cả tools |
| Windows WSL2 | Hoạt động tốt | Quirks networking cho browser (xem Phần 5.4) |
| Windows native | Không hỗ trợ chính thức | Dùng WSL2 |
| Raspberry Pi (ARM64) | Hỗ trợ | Headless only, model local qua Ollama |

---

## 2.2 Cài đặt

```bash
# Cài global qua npm
npm install -g openclaw@latest

# Hoặc pnpm (nhanh hơn)
pnpm add -g openclaw@latest

# Kiểm tra version
openclaw --version
# → openclaw/2026.4.21 node/v24.0.0 linux/x64
```

---

## 2.3 Onboard wizard — giải thích từng bước

```bash
openclaw onboard --install-daemon
```

Wizard đi qua **5 bước**:

### Bước 1 — Gateway setup

Chọn:
- **Port**: mặc định `18789`. Đổi nếu bị conflict.
- **Bind mode**:
  - `loopback` — chỉ localhost (khuyến nghị cho dev/personal)
  - `tailnet` — qua Tailscale VPN (khuyến nghị cho remote access)
  - `any` — public network (⚠️ chỉ khi có firewall cứng + auth token)
- **Auth token**: Wizard tự generate secure token.

### Bước 2 — Model provider

Chọn provider và nhập API key. Key được lưu vào `~/.openclaw/.env`, **không** vào `openclaw.json`.

Providers phổ biến:
- Anthropic (`ANTHROPIC_API_KEY`) — Claude Sonnet/Opus
- OpenAI (`OPENAI_API_KEY`) — GPT-5.x
- Google (`GOOGLE_API_KEY`) — Gemini 2.5 Pro/Flash
- OpenRouter (`OPENROUTER_API_KEY`) — 200+ models, một key

### Bước 3 — Channel setup

Chọn channel đầu tiên. Telegram phổ biến nhất:
1. Tạo bot qua @BotFather → `/newbot`
2. Copy token vào wizard
3. Wizard tự cấu hình webhook

### Bước 4 — Workspace initialization

Wizard tạo `~/.openclaw/workspace/` với template files:
- `SOUL.md` (template nhân cách mặc định)
- `AGENTS.md` (quy tắc mặc định)
- `USER.md` (trống, bạn cần điền)

### Bước 5 — Daemon install (`--install-daemon`)

Tự tạo system service:
- **Linux**: systemd user service (`~/.config/systemd/user/openclaw-gateway.service`)
- **macOS**: launchd plist (`~/Library/LaunchAgents/ai.openclaw.gateway.plist`)

Gateway tự start khi login, restart khi crash.

```bash
# Sau khi onboard
openclaw gateway status    # Kiểm tra đang chạy
openclaw dashboard         # Mở Control UI trên browser
```

---

## 2.4 Cấu trúc thư mục đầy đủ

```
~/.openclaw/
├── openclaw.json              # Main config (không lưu secrets ở đây!)
├── .env                       # API keys — GITIGNORE FILE NÀY!
├── workspace/                 # Agent workspace mặc định
│   ├── SOUL.md
│   ├── IDENTITY.md
│   ├── AGENTS.md
│   ├── USER.md
│   ├── TOOLS.md
│   ├── HEARTBEAT.md
│   ├── MEMORY.md
│   └── memory/                # Daily working notes
│       ├── 2026-04-23.md
│       └── 2026-04-22.md
├── workspaces/                # Multi-agent (nếu dùng)
│   ├── coder/
│   └── assistant/
├── cron/
│   ├── jobs.json              # Cron job definitions
│   └── jobs-state.json        # Runtime state — GITIGNORE!
├── devices/
│   ├── paired.json            # Paired devices
│   └── pending.json           # Pending pair requests
├── skills/                    # Installed skills
│   ├── agent-browser/
│   │   └── SKILL.md
│   └── gog/
│       └── SKILL.md
├── blockrun/                  # ClawRouter data
│   └── wallet.key             # BIP-39 mnemonic — BACKUP VÀ GITIGNORE!
└── canvas/                    # Canvas workspace (macOS)
```

> ⚠️ Không bao giờ commit `~/.openclaw/.env`, `blockrun/wallet.key`, `cron/jobs-state.json` lên git.

---

## 2.5 Lỗi thường gặp khi setup và cách fix

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|---------|
| `pairing required` | Channel yêu cầu device pairing nhưng chưa pair | `openclaw pair --channel telegram` |
| `device_token_mismatch` | `paired.json` lệch với token gateway | Xóa `~/.openclaw/devices/paired.json`, pair lại |
| `gateway closed 1008` | Auth token sai hoặc expire | `openclaw config set gateway.auth.token $(openssl rand -hex 32)` |
| `Cannot find module 'playwright'` | Playwright chưa cài | Xem [Phần 5 — Playwright install](05-browser.md#playwright-install) |
| Port 18789 already in use | Port bị dùng bởi process khác | `openclaw config set gateway.port 19789` |
| `EACCES: permission denied` | npm global install thiếu quyền | Dùng nvm, hoặc `npm config set prefix ~/.npm-global` |
| Telegram bot không reply | Bot chưa được start hoặc allowList sai | Kiểm tra `channels.telegram.allowFrom`, gửi `/whoami` |

### WSL2-specific issues

**Port forwarding cho Extension Relay** (xem chi tiết Phần 5.4):
```powershell
# PowerShell Admin — chạy mỗi lần reboot hoặc thêm vào Task Scheduler
$wslIP = (wsl hostname -I).Trim()
netsh interface portproxy add v4tov4 `
  listenport=18792 listenaddress=127.0.0.1 `
  connectport=18792 connectaddress=$wslIP
```

**Nameserver issue** (DNS không resolve trong WSL2):
```bash
# Trong WSL2
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

---

## 2.6 Quản lý gateway daemon

```bash
# Kiểm tra status
openclaw gateway status

# Restart (sau khi đổi config)
openclaw gateway restart

# Stop / Start thủ công
openclaw gateway stop
openclaw gateway start

# Linux: qua systemd
systemctl --user status openclaw-gateway
systemctl --user restart openclaw-gateway
systemctl --user enable openclaw-gateway  # Auto-start

# macOS: qua launchctl
launchctl list | grep openclaw
```
