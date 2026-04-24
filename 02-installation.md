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

### Cách khuyến nghị — Local prefix installer (macOS / Linux)

Cài đặt openclaw và Node.js cùng nhau dưới `~/.openclaw/`, **không phụ thuộc Node.js hệ thống**:

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

Installer sẽ:
1. Tải Node.js LTS phù hợp về `~/.openclaw/tools/node/`
2. Cài openclaw vào `~/.openclaw/lib/node_modules/`
3. Tạo binary `~/.openclaw/bin/openclaw` — là shell script với dòng `exec` hardcode path Node riêng:
   ```
   exec "/Users/<bạn>/.openclaw/tools/node/bin/node" \
        "/Users/<bạn>/.openclaw/lib/node_modules/openclaw/dist/entry.js" "$@"
   ```

> ⚠️ **Installer không tự thêm PATH trên macOS.** Phải thêm tay:
> ```bash
> echo 'export PATH="$HOME/.openclaw/bin:$PATH"' >> ~/.zshrc
> source ~/.zshrc
> ```

Sau khi cài, kiểm tra:

```bash
openclaw --version
# → openclaw/2026.4.21 node/v24.0.0 darwin/arm64

which openclaw
# → /Users/<bạn>/.openclaw/bin/openclaw
```

> **Lưu ý khi cài Playwright** (cho browser automation): phải dùng npm/npx của openclaw, không phải hệ thống:
> ```bash
> # Cài Playwright vào node_modules của openclaw
> ~/.openclaw/tools/node/bin/npm install -g playwright
>
> # Tải Chromium (đủ cho hầu hết tác vụ)
> ~/.openclaw/tools/node/bin/npx playwright install chromium
>
> # Kiểm tra
> ~/.openclaw/tools/node/bin/npx playwright --version
> ```
> Không dùng `npm`/`npx` hệ thống — openclaw tìm module theo NODE_PATH riêng, Playwright cài ngoài sẽ không thấy.

### Cách thay thế — npm global (nếu đã có Node.js hệ thống)

```bash
npm install -g openclaw@latest
# hoặc
pnpm add -g openclaw@latest
```

> Cách này để openclaw không phụ thuộc vào Node.js hệ thống. Nếu Node bị cập nhật hoặc xoá, openclaw có thể bị ảnh hưởng.

---

## 2.3 Phiên bản tháng 4/2026 — Thay đổi quan trọng cần biết

> Áp dụng cho ai đang nâng cấp từ bản cũ hơn `2026.4.10`. Người cài mới lần đầu bỏ qua mục này.

### Breaking changes — config cũ sẽ không hoạt động

| Cái cũ | Thay bằng | Ghi chú |
|--------|-----------|---------|
| Env var `OPENCLAW_CODEX_APP_SERVER_GUARDIAN` | `plugins.entries.codex.config.appServer.mode: "guardian"` | Env var bị xoá hoàn toàn |
| Profile `legacy.openai-codex:default` | Identity-scoped profile | Chạy `openclaw doctor` để dọn |
| `~/.codex` OAuth import trong onboarding | Browser login / device pairing | Path cũ bị block |
| `jobs.json` chứa cả runtime state | `jobs.json` (định nghĩa) + `jobs-state.json` (runtime) | Phải gitignore `jobs-state.json` |
| `models.json` per-agent | `models.json` chung duy nhất | Không bị ghi đè khi chạy `models list` |

### Config keys mới thêm trong tháng 4/2026

```jsonc
// ~/.openclaw/openclaw.json
{
  "channels": {
    "telegram": {
      "pollingStallThresholdMs": 120000   // watchdog timeout polling (mặc định 120s)
    },
    "bluebubbles": {
      "sendTimeoutMs": 30000              // timeout gửi iMessage
    }
  },
  "agents": {
    "defaults": {
      "imageModel": "gpt-image-2",        // model ảnh mặc định trước khi vision skip
      "promptOverlays": {
        "gpt5": { "personality": "balanced" }
      }
    }
  },
  "memory": {
    "canonicalFile": "MEMORY.md"          // luôn viết hoa, không phải memory.md
  }
}
```

### Thinking mode — thay đổi hành vi

- Khi đổi provider, setting `max` reasoning **tự remap** sang mode lớn nhất provider mới hỗ trợ — không cần sửa tay.
- `/think off` trên model có adapter giới hạn sẽ thực sự tắt, không còn bật ngầm lên `high`.

### Security hardening — chú ý nếu dùng `.env` workspace

```bash
# Các biến này giờ bị BLOCK trong .env của workspace (untrusted):
OPENCLAW_*          # toàn bộ biến OPENCLAW_ env
MINIMAX_API_HOST    # block URL injection
NODE_OPTIONS        # block trong MCP stdio servers
```

API keys provider (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, ...) trong `~/.openclaw/.env` vẫn hoạt động bình thường — chỉ workspace `.env` bị restrict.

### Tính năng mới đáng chú ý (2026.4.10 → 2026.4.22)

| Tính năng | Version | Ghi chú |
|-----------|---------|---------|
| xAI image generation + TTS/STT | 2026.4.22 | Model `grok-imagine-image`, 6 giọng live |
| OpenAI Realtime voice + agent handoff | 2026.4.10 | Gateway tự mint ephemeral secret |
| Google Meet plugin | 2026.4.10 | Chrome/Twilio transport, personal auth |
| LanceDB vector search (KNN) | 2026.4.10 | Tích hợp memory system |
| `openclaw qa suite` fail by default | 2026.4.20 | CI scripts cần thêm `--allow-failures` nếu muốn giữ behavior cũ |
| Session export `/export-trajectory` | 2026.4.10 | Bundle transcript + artifacts đã redact |

### Dọn dẹp sau khi nâng cấp

```bash
# Chạy doctor để openclaw tự phát hiện config lỗi thời
openclaw doctor

# Nếu thấy cảnh báo "legacy profile", xoá bằng:
openclaw config cleanup --legacy-profiles

# Đảm bảo jobs-state.json đã gitignore
echo "jobs-state.json" >> ~/.openclaw/.gitignore
```

---

## 2.4 Onboard wizard — giải thích từng bước

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

## 2.5 Cấu trúc thư mục đầy đủ

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

## 2.6 Lỗi thường gặp khi setup và cách fix

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

## 2.7 Quản lý gateway daemon

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

---

## 2.8 Gỡ cài đặt & dọn dẹp hoàn toàn

### Bước 1 — Dừng và xoá daemon

**macOS (launchd):**
```bash
# Unload và xoá plist
launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>/dev/null
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

**Linux (systemd):**
```bash
systemctl --user stop openclaw-gateway
systemctl --user disable openclaw-gateway
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Bước 2 — Gỡ package npm / pnpm

```bash
# Kiểm tra bản đang dùng trước
which openclaw
# → nếu thấy path chứa npm-packages hoặc node_modules thì dùng npm uninstall

# npm
npm uninstall -g openclaw

# pnpm
pnpm remove -g openclaw

# Xác nhận đã gỡ
which openclaw   # → không còn output
```

### Bước 3 — Xoá toàn bộ dữ liệu người dùng

> ⚠️ Bước này xoá **vĩnh viễn** config, API keys, workspace, memory, wallet. Backup trước nếu cần.

```bash
# Backup trước nếu muốn giữ lại
cp -r ~/.openclaw ~/openclaw-backup-$(date +%Y%m%d)

# Xoá thư mục state chính
rm -rf ~/.openclaw

# Xoá profile dev (nếu đã dùng --dev)
rm -rf ~/.openclaw-dev

# Xoá profile tuỳ chỉnh khác (nếu có)
# rm -rf ~/.openclaw-<tên-profile>
```

### Bước 4 — Xoá log và cache hệ thống (macOS)

```bash
# Log macOS
rm -rf ~/Library/Logs/openclaw* 2>/dev/null

# Application Support (nếu có)
rm -rf ~/Library/Application\ Support/openclaw 2>/dev/null

# Preferences (nếu có plist)
rm -f ~/Library/Preferences/ai.openclaw.* 2>/dev/null
```

### Kiểm tra đã sạch

```bash
which openclaw          # → không có output
ls ~/.openclaw 2>&1     # → No such file or directory
npm list -g openclaw 2>&1 | grep openclaw  # → không có output
```

### Bảng tóm tắt file/thư mục cần xoá

| Đường dẫn | Nội dung | Gỡ ở bước |
|-----------|---------|-----------|
| `~/Library/LaunchAgents/ai.openclaw.gateway.plist` | Daemon macOS | 1 |
| `~/.config/systemd/user/openclaw-gateway.service` | Daemon Linux | 1 |
| `~/.openclaw/` | Config, keys, workspace, memory, wallet | 3 |
| `~/.openclaw-dev/` | State profile dev | 3 |
| `~/Library/Logs/openclaw*` | Log macOS | 4 |
| `~/Library/Application Support/openclaw` | App support macOS | 4 |

---

## 2.9 Cài xong → chạy gì? Các lệnh CLI cơ bản

### Luồng khởi động chuẩn (lần đầu sau cài đặt)

```bash
# 1. Chạy onboarding (chỉ làm một lần)
openclaw onboard --install-daemon

# 2. Kiểm tra gateway đang chạy
openclaw gateway status

# 3. Mở Control UI trên trình duyệt
openclaw dashboard
```

Nếu đã onboard rồi và chỉ muốn khởi động lại:

```bash
openclaw gateway start      # khởi động gateway
openclaw gateway restart    # restart (áp dụng config mới)
openclaw gateway stop       # dừng gateway
```

---

### Bảng lệnh CLI cơ bản

| Nhóm | Lệnh | Mô tả |
|------|------|-------|
| **Kiểm tra** | `openclaw --version` | Xem phiên bản đang chạy |
| | `openclaw gateway status` | Gateway đang chạy / dừng, port, uptime |
| | `openclaw doctor` | Tự chẩn đoán config, phát hiện lỗi thường gặp |
| **Khởi động** | `openclaw gateway start` | Khởi động gateway thủ công |
| | `openclaw gateway stop` | Dừng gateway |
| | `openclaw gateway restart` | Restart gateway (bắt buộc sau khi sửa config) |
| **Giao diện** | `openclaw dashboard` | Mở Control UI tại `http://localhost:18789` |
| | `openclaw logs` | Xem log real-time của gateway |
| | `openclaw logs --tail 100` | Xem 100 dòng log cuối |
| **Cấu hình** | `openclaw config get <key>` | Đọc một key trong `openclaw.json` |
| | `openclaw config set <key> <value>` | Ghi một key vào `openclaw.json` |
| | `openclaw config edit` | Mở `openclaw.json` bằng `$EDITOR` |
| **Cập nhật** | `openclaw update` | Cập nhật lên phiên bản mới nhất |
| | `openclaw update --check` | Chỉ kiểm tra có bản mới, không cài |
| **Kết nối** | `openclaw pair --channel telegram` | Ghép cặp Telegram bot |
| | `openclaw pair --channel bluebubbles` | Ghép cặp BlueBubbles (iMessage) |
| | `openclaw pair --list` | Danh sách device đã pair |
| **Models** | `openclaw models list` | Liệt kê models có sẵn theo provider |
| | `openclaw models set <model-id>` | Đặt model mặc định |
| **Onboarding** | `openclaw onboard` | Chạy lại wizard (không ghi đè config cũ) |
| | `openclaw onboard --install-daemon` | Chạy wizard và cài systemd/launchd |
| **Dọn dẹp** | `openclaw config cleanup --legacy-profiles` | Xoá profiles cũ không còn dùng |

---

### Workflow thường gặp

**Sau khi sửa `openclaw.json`:**
```bash
openclaw config edit          # sửa config
openclaw gateway restart      # bắt buộc để áp dụng
openclaw gateway status       # xác nhận đang chạy
```

**Khi gateway không phản hồi:**
```bash
openclaw doctor               # chẩn đoán
openclaw logs --tail 50       # xem log lỗi gần nhất
openclaw gateway restart      # thử restart
```

**Kiểm tra nhanh mọi thứ OK:**
```bash
openclaw --version && openclaw gateway status && openclaw doctor
```

---

## 2.10 Native Desktop App — có cần cài openclaw daemon không?

### Phân biệt hai chế độ triển khai

| Chế độ | Cài daemon? | Gateway do ai quản lý? |
|--------|-------------|------------------------|
| **CLI + daemon** (cài qua npm/installer) | **Bắt buộc** nếu muốn tự start khi login | `systemd` (Linux) hoặc `launchd` (macOS) — bạn tự cài qua `onboard --install-daemon` |
| **Native Desktop App** (file `.dmg` / `.AppImage`) | **Không cần** | App tự khởi động và quản lý gateway embedded bên trong — tắt app là tắt gateway |

---

### Native Desktop App — những điểm cần biết

**Khi mở app:**
- Gateway tự start ngầm, không cần chạy lệnh nào thêm.
- `openclaw gateway status` vẫn hoạt động và sẽ hiện trạng thái của gateway do app quản lý.
- Không cần (và không nên) chạy `openclaw onboard --install-daemon` — nếu chạy sẽ tạo thêm một daemon song song, gây conflict port.

**Khi đóng app:**
- Gateway dừng theo. Nếu muốn gateway chạy ngầm sau khi đóng cửa sổ → bật tùy chọn **"Run in background"** / **"Keep gateway alive on close"** trong Settings của app.

**Khi cần dùng CLI song song với app:**
```bash
# CLI vẫn dùng được bình thường, trỏ vào cùng gateway
openclaw dashboard       # mở UI
openclaw config set ...  # sửa config (nhớ restart gateway trong app sau khi sửa)
openclaw logs            # xem log
```

**Cập nhật:**
- Native app tự kiểm tra bản mới khi khởi động, cập nhật qua menu **Help → Check for Updates**.
- Lệnh `openclaw update` chỉ có tác dụng với bản cài qua npm/installer, **không** cập nhật native app.

> **Tóm lại:** Dùng Native Desktop App → mở app là xong, không cần quan tâm daemon. Chỉ cài daemon khi dùng CLI thuần và muốn gateway tự chạy khi login mà không cần mở app.

---

## 2.11 Cài đặt SearXNG (self-hosted search engine)

SearXNG là search engine tự host, cho phép openclaw tìm kiếm web mà không cần API key của Google/Bing. Cài qua Docker.

### Yêu cầu

- Docker Desktop đã cài và đang chạy
- `openssl` có sẵn trên máy (macOS có sẵn)

### Bước 1 — Pull Docker image

```bash
docker pull searxng/searxng
```

### Bước 2 — Tạo thư mục config

```bash
mkdir -p ~/.searxng
```

> ⚠️ Phải tạo thư mục trước. `docker run` không tự tạo thư mục host.

### Bước 3 — Tạo file `settings.yml`

Dùng `printf` thay vì heredoc để tránh lỗi **non-breaking space** (`\xa0`) khi copy-paste từ trình duyệt — YAML không nhận `\xa0` làm indentation và sẽ crash:

```bash
SECRET=$(openssl rand -hex 32) && printf 'use_default_settings:\n  engines:\n    keep_only:\n      - duckduckgo\n      - bing\n      - qwant\n\nserver:\n  secret_key: "%s"\n  limiter: false\n  image_proxy: false\n\nsearch:\n  formats:\n    - html\n    - json\n' "$SECRET" > ~/.searxng/settings.yml
```

Kiểm tra file hợp lệ:

```bash
cat ~/.searxng/settings.yml
# → Phải thấy secret_key là chuỗi hex 64 ký tự
```

### Bước 4 — Chạy container

```bash
docker run -d \
  --name searxng \
  -p 8889:8080 \
  -v ~/.searxng:/etc/searxng \
  searxng/searxng
```

### Bước 5 — Kiểm tra

```bash
# Xem log (chờ vài giây sau khi chạy)
docker logs searxng

# Thử truy cập
curl -s "http://localhost:8889/search?q=test&format=json" | head -c 200
```

Nếu trả về JSON là SearXNG đang hoạt động. Mở trình duyệt vào `http://localhost:8889` để dùng giao diện web.

### Quản lý container

```bash
# Dừng
docker stop searxng

# Khởi động lại
docker start searxng

# Xem log real-time
docker logs -f searxng

# Xoá container (giữ nguyên config trong ~/.searxng)
docker rm searxng
```

### Lỗi thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|---------|
| `yaml.scanner.ScannerError` line 4 col 1 | File `settings.yml` chứa non-breaking space (`\xa0`) do copy-paste | Dùng lệnh `printf` ở Bước 3 để tạo lại file |
| `no such file or directory: ~/.searxng/settings.yml` | Thư mục `~/.searxng` chưa tồn tại | Chạy `mkdir -p ~/.searxng` trước |
| `port is already allocated` | Port 8889 đang bị dùng bởi process khác | Đổi `-p 8889:8080` thành `-p 8890:8080` |
| Container exit ngay sau khi chạy | Xem log: `docker logs searxng` | Thường do YAML lỗi hoặc `secret_key` trống |

### Cấu hình trong openclaw

Sau khi SearXNG chạy, thêm vào `~/.openclaw/openclaw.json`:

```jsonc
{
  "tools": {
    "search": {
      "provider": "searxng",
      "url": "http://localhost:8889"
    }
  }
}
```

Sau đó restart gateway:

```bash
openclaw gateway restart
```

---

## 2.12 Cài đặt Skills — Dependencies và cách fix lỗi

Skills trong openclaw là các module mở rộng khả năng của agent (tóm tắt văn bản, đọc PDF, xem video frame, quản lý GitHub issues...). Mỗi skill có thể yêu cầu thêm tool hệ thống — thiếu tool đó thì skill install hoặc runtime sẽ fail.

### Skills bundled vs ClawHub

Có hai loại skills:

| Loại | Nguồn | Cách activate |
|------|-------|---------------|
| **openclaw-bundled** | Có sẵn trong openclaw, không cần cài thêm | Chỉ cần cài **system dependency** → tự chuyển `✓ ready` |
| **ClawHub** | Tải từ clawhub.com | Dùng `openclaw skills install <tên>` |

Hầu hết skills mặc định (summarize, nano-pdf, tmux, github...) đều là **openclaw-bundled** — đã có sẵn, chỉ cần cài đúng tool hệ thống là dùng được ngay.

```bash
# Xem danh sách skills và trạng thái ready / needs setup
openclaw skills list

# Xem skills nào còn thiếu dependency
openclaw skills check

# Xem chi tiết yêu cầu của một skill
openclaw skills info <tên-skill>

# Chỉ dùng lệnh này cho ClawHub skills (KHÔNG dùng cho bundled):
openclaw skills install <tên-skill>
```

---

### 8 Built-in Skills trên Dashboard — Fix orange dot

Dashboard **Skills → BUILT-IN SKILLS** hiển thị 8 skills mặc định. Skill nào thiếu dependency sẽ hiện **chấm cam**.

Chạy lệnh này để thấy chính xác skill nào thiếu gì:

```bash
openclaw skills check
```

Output mẫu:

```
Missing requirements:
  apple-notes  (bins: memo)
  blogwatcher  (bins: blogwatcher)
  bluebubbles  (config: channels.bluebubbles)
  goplaces     (bins: goplaces; env: GOOGLE_PLACES_API_KEY)
  himalaya     (bins: himalaya)
  model-usage  (bins: codexbar)
```

Bảng tóm tắt:

| Skill | Cần gì | Loại |
|-------|--------|------|
| `apple-notes` | binary `memo` | npm CLI |
| `blogwatcher` | binary `blogwatcher` | npm CLI |
| `bluebubbles` | config `channels.bluebubbles` | JSON config (không cần binary) |
| `goplaces` | binary `goplaces` + env `GOOGLE_PLACES_API_KEY` | npm CLI + API key |
| `himalaya` | binary `himalaya` | brew |
| `model-usage` | binary `codexbar` | npm CLI |
| `session-logs` | binary `jq` | brew |
| `summarize` | binary `summarize` | npm CLI (xem [fix bên dưới](#fix-lỗi-summarize-skill--brew-formula-bị-xoá)) |

---

#### Lưu ý quan trọng về npm PATH

OpenClaw chạy qua launchd với PATH riêng — **không giống** PATH trong terminal của bạn. Nếu cài npm package bằng `npm install -g` thông thường, binary có thể vào `~/.npm-packages/bin/` hoặc `/usr/local/lib/` — những path này OpenClaw **không thấy**.

Kiểm tra npm prefix hiện tại:

```bash
npm config get prefix
```

Nếu **không phải** `~/.npm-global`, đổi lại:

```bash
npm config set prefix ~/.npm-global
```

Sau đó mọi `npm install -g` sẽ đưa binary vào `~/.npm-global/bin/` — path này **đã có sẵn** trong launchd PATH của OpenClaw.

Xác nhận lại:

```bash
npm config get prefix   # → /Users/<bạn>/.npm-global
```

---

#### Cài tất cả skills trong một lần (macOS)

```bash
# Bước 1 — Đảm bảo npm prefix đúng
npm config set prefix ~/.npm-global

# Bước 2 — Cài CLIs qua npm (sẽ vào ~/.npm-global/bin/)
npm install -g blogwatcher goplaces codexbar

# Bước 3 — Cài CLIs qua brew (/opt/homebrew/bin/ đã có trong PATH)
brew install himalaya jq

# Bước 4 — Cài memo cho apple-notes
npm install -g @apple-notes-mcp/memo

# Bước 5 — Cài summarize
npm install -g @steipete/summarize

# Bước 6 — Xác nhận
which blogwatcher && which goplaces && which codexbar
which himalaya && which jq && which memo && which summarize

# Bước 7 — Restart gateway để nhận PATH mới
openclaw gateway restart

# Bước 8 — Kiểm tra kết quả
openclaw skills check
```

---

#### bluebubbles — Không cần binary, chỉ cần config

`openclaw skills check` báo `config: channels.bluebubbles` — skill này **không cần cài CLI gì cả**, chỉ cần config JSON.

BlueBubbles yêu cầu **một máy Mac luôn bật** chạy [BlueBubbles server app](https://bluebubbles.app):

1. Cài BlueBubbles server trên Mac → mở app → Settings → lấy **Server URL** và **Server Password**
2. Thêm vào `~/.openclaw/openclaw.json`:

```jsonc
{
  "channels": {
    "bluebubbles": {
      "url": "http://your-mac-ip:1234",
      "password": "your-server-password"
    }
  }
}
```

3. Pair và restart:

```bash
openclaw pair --channel bluebubbles
openclaw gateway restart
```

---

#### goplaces — Cần thêm API key

Sau khi cài binary `goplaces`, phải thêm API key vào `~/.openclaw/.env`:

```bash
echo 'GOOGLE_PLACES_API_KEY=AIza...' >> ~/.openclaw/.env
```

Lấy key tại: `console.cloud.google.com` → **APIs & Services** → **Credentials** → **Create API Key** → enable **Places API (New)**.

---

#### himalaya — Cần config tài khoản sau khi cài

```bash
brew install himalaya      # macOS
# Linux: curl -sSL https://github.com/pimalaya/himalaya/releases/latest/download/himalaya-x86_64-unknown-linux-musl.tar.gz | tar xz -C /usr/local/bin
```

Config tài khoản email (wizard tương tác):

```bash
himalaya account configure
```

Hoặc tạo thủ công `~/.config/himalaya/config.toml`:

```toml
[accounts.gmail]
default = true
email = "you@gmail.com"
display-name = "Your Name"

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption = "ssl"
backend.login = "you@gmail.com"
backend.auth.type = "password"
backend.auth.raw = "your-app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption = "tls"
message.send.backend.login = "you@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.raw = "your-app-password"
```

> Gmail: dùng **App Password** (không phải mật khẩu thường). Bật tại `myaccount.google.com → Security → 2-Step Verification → App passwords`.

---

### Bảng dependencies của từng skill phổ biến

| Skill | Dùng để làm gì | System dependencies cần cài | Ghi chú |
|-------|----------------|----------------------------|---------|
| `nano-pdf` | Đọc / trích xuất text từ PDF | `poppler` (cung cấp `pdftotext`, `pdfinfo`) | Thiếu poppler → skill fail hoàn toàn |
| `summarize` | Tóm tắt file/text dài | binary `summarize` qua npm | Brew formula bị xoá — xem [fix bên dưới](#fix-lỗi-summarize-skill--brew-formula-bị-xoá) |
| `session-logs` | Parse và query log JSON | `jq` | |
| `video-frames` | Trích xuất frame từ video | `ffmpeg` | File video lớn cần disk space |
| `github` | Tương tác GitHub repo | `gh` (GitHub CLI) | Cần auth: `gh auth login` |
| `gh-issues` | Quản lý GitHub Issues | `gh` (GitHub CLI) | Dùng chung auth với `github` skill |
| `tmux` | Tương tác terminal session | `tmux` | |
| `bear-notes` | Đọc / ghi Bear notes | Bear app (macOS App Store) | macOS only |
| `clawhub` | Kết nối ClawHub marketplace | Không cần dependency hệ thống | Cần ClawHub account |
| `agent-browser` | Điều khiển trình duyệt | Playwright + Chromium — xem [Phần 5](05-browser.md) | |

---

### Cài dependencies theo OS

#### macOS (Homebrew)

```bash
# nano-pdf
brew install poppler

# summarize — brew formula BỊ XOÁ, cài qua npm (xem mục "Fix lỗi summarize" bên dưới)
# Không cần brew install gì thêm cho summarize

# session-logs
brew install jq

# video-frames
brew install ffmpeg

# github + gh-issues
brew install gh
gh auth login               # xác thực GitHub

# tmux
brew install tmux
```

#### Linux — Ubuntu / Debian

```bash
sudo apt update

# nano-pdf
sudo apt install -y poppler-utils

# summarize
sudo apt install -y ripgrep curl

# session-logs
sudo apt install -y jq

# video-frames
sudo apt install -y ffmpeg

# github + gh-issues
sudo apt install -y gh
# hoặc nếu apt không có sẵn gh:
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install -y gh
gh auth login

# tmux
sudo apt install -y tmux
```

#### Linux — Arch

```bash
sudo pacman -S --noconfirm poppler ripgrep curl jq ffmpeg github-cli tmux
gh auth login
```

---

### Fix lỗi: `summarize` skill — brew formula bị xoá

Từ commit `93870cb` trên `steipete/tap`, formula `steipete/tap/summarize` đã bị xoá. Cài bằng brew sẽ fail:

```
Warning: No available formula or cask with the name "steipete/tap/summarize"
steipete/tap/summarize was deleted from steipete/tap in commit 93870cb
```

`summarize` thực chất là tool [`@steipete/summarize`](https://github.com/steipete/summarize) — prebuilt binary, phân phối qua npm (hoặc GitHub Releases trực tiếp).

**macOS — cài binary từ GitHub Releases:**

```bash
# Apple Silicon (arm64)
curl -fsSL https://github.com/steipete/summarize/releases/download/v0.13.0/summarize-macos-arm64-v0.13.0.tar.gz \
  | tar -xz -C /usr/local/bin

# Intel (x64)
curl -fsSL https://github.com/steipete/summarize/releases/download/v0.13.0/summarize-macos-x64-v0.13.0.tar.gz \
  | tar -xz -C /usr/local/bin

chmod +x /usr/local/bin/summarize
summarize --version
```

**macOS / Linux — cài qua npm của openclaw (khuyến nghị nếu dùng local prefix installer):**

```bash
# Dùng npm bundled trong openclaw để summarize nằm đúng NODE_PATH
~/.openclaw/tools/node/bin/npm install -g @steipete/summarize

# Kiểm tra
~/.openclaw/tools/node/bin/npx summarize --version
```

**Linux — cài qua npm hệ thống:**

```bash
npm install -g @steipete/summarize
summarize --version
```

> Nếu không muốn cài ngay, có thể bỏ qua — các skill khác không phụ thuộc vào nó. `openclaw doctor` sẽ báo "missing requirement" nhưng không ảnh hưởng đến gateway.

**Cách bỏ qua skill `summarize` trong wizard onboard:**

Khi wizard hiện danh sách "Install missing skill dependencies", bỏ chọn `summarize` bằng phím **Space** trước khi nhấn Enter.

---

### Kiểm tra sau khi cài dependencies

```bash
# Kiểm tra từng tool đã có chưa
which pdftotext && pdftotext -v     # nano-pdf
which summarize && summarize --version  # summarize
which jq && jq --version            # session-logs
which ffmpeg && ffmpeg -version     # video-frames
which gh && gh --version            # github / gh-issues
which tmux && tmux -V               # tmux

# Xem skills nào đã ready sau khi cài dependencies
openclaw skills list

# Xem chi tiết skills còn thiếu gì
openclaw skills check

# Doctor tổng quát
openclaw doctor
```

> Với **openclaw-bundled skills**: không cần chạy `openclaw skills install` — cài system dependency xong là skill tự chuyển sang `✓ ready`.

### Lỗi thường gặp khi cài skill

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|---------|
| `Install failed: summarize (exit 1)` | Brew formula bị xoá, skill là bundled nên không dùng `skills install` | Cài qua `~/.openclaw/tools/node/bin/npm install -g @steipete/summarize` |
| `pdftotext: command not found` | Thiếu poppler | `brew install poppler` / `apt install poppler-utils` |
| `Missing requirements: 44` khi onboard | Chưa cài system dependencies | Cài theo bảng ở trên rồi chạy lại `openclaw skills install --all` |
| `gh: command not found` | Thiếu GitHub CLI | `brew install gh` rồi `gh auth login` |
| Skill install OK nhưng vẫn lỗi lúc dùng | Runtime thiếu lib | `openclaw skills info <tên>` để xem full requirement list |
