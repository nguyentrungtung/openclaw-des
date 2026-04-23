# QUICKSTART — Từ Zero đến Chạy Được (15 phút)

---

## Bước 1 — Cài Node.js 24 (2 phút)

```bash
# macOS (Homebrew)
brew install node@24
echo 'export PATH="/usr/local/opt/node@24/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Ubuntu/Debian (nvm — khuyến nghị)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 24
nvm use 24

# Verify
node --version   # → v24.x.x
npm --version    # → 10.x.x
```

---

## Bước 2 — Cài OpenClaw (1 phút)

```bash
npm install -g openclaw@latest
openclaw --version   # → openclaw/2026.4.21
```

---

## Bước 3 — Onboard wizard (5 phút)

```bash
openclaw onboard --install-daemon
```

Wizard hỏi lần lượt:

1. **Port**: nhấn Enter giữ mặc định (18789)
2. **Bind mode**: chọn `loopback` (nhấn Enter)
3. **Model provider**: chọn Anthropic hoặc OpenRouter
4. **API key**: paste vào
5. **Channel**: chọn Telegram
6. **Bot token**: paste token từ @BotFather

Tạo bot Telegram nếu chưa có:
- Mở Telegram → tìm `@BotFather`
- Gửi `/newbot` → đặt tên → copy token

---

## Bước 4 — Điền USER.md (3 phút)

```bash
# Mở file để điền thông tin của bạn
nano ~/.openclaw/workspace/USER.md
# Hoặc dùng bất kỳ editor nào
```

Điền tối thiểu: tên, timezone, ngôn ngữ ưa thích, stack kỹ thuật.

---

## Bước 5 — Cài Playwright (2 phút)

```bash
cd $(npm root -g)/openclaw
npm install playwright --legacy-peer-deps
npx playwright install chromium

openclaw gateway restart
openclaw doctor --fix   # Phải thấy "playwright: ok"
```

---

## Bước 6 — Cài ClawRouter (1 phút)

```bash
curl -fsSL https://raw.githubusercontent.com/BlockRunAI/ClawRouter/main/scripts/reinstall.sh | bash
openclaw gateway restart
```

---

## Bước 7 — Test (2 phút)

```bash
# Lấy Telegram User ID của bạn
# → Gửi /whoami cho bot → bot trả về ID

# Thêm ID vào allowlist
openclaw config set channels.telegram.allowFrom '["YOUR_TELEGRAM_USER_ID"]'
openclaw gateway restart
```

Trong Telegram, gửi cho bot:
- `Hello, bạn là ai?` — phải nhận được response
- `/model auto` — bật smart routing (nếu đã cài ClawRouter)
- `Mở https://example.com và cho tôi biết title của trang` — test browser

---

## Checklist hoàn thành

```
✅ openclaw --version      → 2026.4.x
✅ openclaw gateway status → running
✅ openclaw doctor --fix   → playwright: ok
✅ Telegram bot trả lời
✅ Browser snapshot hoạt động
✅ (optional) /model auto hoạt động → ClawRouter smart routing
```

---

## Bước tiếp theo

1. **Đọc workspace templates** → [10-workspace-templates.md](10-workspace-templates.md)
   - Chỉnh SOUL.md, AGENTS.md theo nhu cầu
   - Seed MEMORY.md với facts ban đầu

2. **Setup automation** → [04-automation.md](04-automation.md)
   - Thêm morning briefing cron
   - Bật heartbeat monitoring

3. **Cài skills phổ biến** → [06-skills-plugins.md](06-skills-plugins.md)
   ```bash
   openclaw skills install agent-browser
   openclaw skills install gog           # Nếu dùng Google Workspace
   openclaw skills install github        # Nếu dùng GitHub
   ```

4. **Hardening** → [08-security.md](08-security.md)
   ```bash
   openclaw security audit --fix
   ```

---

## Lệnh hay dùng nhất

```bash
# Gateway
openclaw gateway status
openclaw gateway restart
openclaw dashboard              # Mở Control UI

# Config
openclaw config get agents.defaults.model.primary
openclaw config set channels.telegram.allowFrom '["ID"]'

# Browser
openclaw browser start
openclaw browser open https://example.com
openclaw browser snapshot

# Cron
openclaw cron list
openclaw cron run <job-id>      # Test job ngay

# Skills
openclaw skills search "keyword"
openclaw skills install <slug>
openclaw skills list

# Logs
openclaw logs --follow
openclaw doctor --fix
```

---

## Troubleshooting nhanh

| Vấn đề | Lệnh debug |
|--------|-----------|
| Bot không trả lời | `openclaw logs --follow` + gửi message |
| Snapshot lỗi 501 | `openclaw doctor --fix` |
| Gateway không start | `openclaw logs --level error` |
| Cron không chạy | `openclaw cron status`, kiểm tra timezone |
| ClawRouter không hoạt động | `openclaw gateway status`, xem ClawRouter plugin |
