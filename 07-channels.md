# Phần 7: Channels & Communication

---

## 7.1 Telegram — setup chi tiết

Telegram là channel phổ biến nhất cho OpenClaw vì dễ setup và có app tốt trên mọi platform.

### Bước 1: Tạo bot

1. Mở Telegram → tìm `@BotFather`
2. Gửi `/newbot`
3. Đặt tên hiển thị (vd: "My AI Assistant")
4. Đặt username (phải kết thúc bằng `bot`, vd: `myassistant_bot`)
5. Copy **bot token** (dạng `1234567890:AAFxxxxxxxxxxxxxxxxxxxxxxxx`)

### Bước 2: Lấy Telegram User ID của bạn

```
# Gửi /whoami cho bot sau khi config xong
# Hoặc gửi bất kỳ message nào, openclaw log sẽ hiển thị user ID
```

Cách khác: gửi `/start` cho `@userinfobot` để lấy ID của bạn.

### Bước 3: Config

```bash
# Bot token (lưu vào .env)
echo 'TELEGRAM_BOT_TOKEN=1234567890:AAFxxxxxx' >> ~/.openclaw/.env

# Config trong openclaw.json
openclaw config set channels.telegram.enabled true
openclaw config set channels.telegram.botToken '$TELEGRAM_BOT_TOKEN'

# Bảo mật: chỉ cho phép user ID của bạn
openclaw config set channels.telegram.dmPolicy "allowlist"
openclaw config set channels.telegram.allowFrom '["YOUR_TELEGRAM_USER_ID"]'

# Group settings (chỉ respond khi được @mention)
openclaw config set channels.telegram.groupPolicy "mention"

# Typing indicator (stream message từng phần)
openclaw config set channels.telegram.streamMode "partial"

# Restart để apply
openclaw gateway restart
```

### Config đầy đủ trong openclaw.json

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "{{ từ .env }}",
      "dmPolicy": "allowlist",
      "allowFrom": ["123456789"],
      "groupPolicy": "mention",
      "streamMode": "partial",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  }
}
```

### dmPolicy options

| Giá trị | Ý nghĩa |
|---------|---------|
| `open` | Bất kỳ ai nhắn được (⚠️ nguy hiểm) |
| `allowlist` | Chỉ IDs trong `allowFrom` (**khuyến nghị**) |
| `pairing` | Cần device pairing trước |

### groupPolicy options

| Giá trị | Ý nghĩa |
|---------|---------|
| `open` | Respond mọi message trong group |
| `mention` | Chỉ respond khi @mention bot (**khuyến nghị**) |
| `disabled` | Không respond trong group |

---

## 7.2 Discord setup

```bash
# 1. Tạo bot tại discord.com/developers/applications
# 2. Enable "Message Content Intent" trong Bot settings
# 3. Copy bot token
# 4. Invite bot vào server với permissions: Send Messages, Read Message History

openclaw channels login --channel discord
# Wizard sẽ yêu cầu Discord bot token

# Config
openclaw config set channels.discord.enabled true
openclaw config set channels.discord.token '$DISCORD_BOT_TOKEN'
```

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "{{ DISCORD_BOT_TOKEN }}",
      "allowFrom": ["YOUR_DISCORD_USER_ID"],
      "servers": {
        "YOUR_SERVER_ID": {
          "enabled": true,
          "requireMention": true
        }
      }
    }
  }
}
```

---

## 7.3 WhatsApp setup

WhatsApp phức tạp hơn do policy của Meta.

```bash
# Login flow (scan QR code)
openclaw channels login --channel whatsapp
# → Hiện QR code trong terminal
# → Scan bằng WhatsApp trên điện thoại: Settings → Linked Devices → Link Device
```

> ⚠️ WhatsApp không cung cấp official API cho cá nhân. OpenClaw dùng reverse-engineered protocol. Tài khoản có thể bị ban nếu vi phạm ToS của WhatsApp.

---

## 7.4 Signal setup

```bash
openclaw channels login --channel signal
# → Cần Signal CLI hoặc signal-api server chạy local
```

Signal yêu cầu setup phức tạp hơn vì cần proxy server. Xem [docs.openclaw.ai/channels/signal](https://docs.openclaw.ai/channels/signal) để biết chi tiết.

---

## 7.5 iMessage (macOS only)

```bash
# Chỉ hoạt động trên macOS với iCloud account
openclaw channels login --channel imessage
```

iMessage integration dùng AppleScript, yêu cầu:
- macOS với Messages app đã login iCloud
- Accessibility permissions cho openclaw

---

## 7.6 Channel management

```bash
# Xem status tất cả channels
openclaw channels status
openclaw channels list

# Enable/disable channel
openclaw config set channels.telegram.enabled false
openclaw config set channels.discord.enabled true
openclaw gateway restart

# Test channel (gửi message test)
openclaw channels test --channel telegram --to "YOUR_USER_ID" --message "Test message"
```

---

## 7.7 Multi-channel routing

Một Gateway với nhiều channels → một agent trả lời qua tất cả:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "allowFrom": ["TG_USER_ID"]
    },
    "discord": {
      "enabled": true,
      "allowFrom": ["DC_USER_ID"]
    }
  }
}
```

Session key được derive từ channel + user ID, nên conversation Telegram và Discord là **2 sessions riêng biệt** (context không share).

Để share context giữa channels, cần config explicit session key:

```bash
# Gửi message với session key cố định
openclaw agent \
  --session-id "main-personal" \
  --message "Nhắc tôi họp lúc 3h chiều" \
  --deliver \
  --channel telegram
```

---

## 7.8 Channels được hỗ trợ (đầy đủ)

| Channel | Status | Ghi chú |
|---------|--------|---------|
| Telegram | Stable ✅ | Phổ biến nhất, setup đơn giản |
| Discord | Stable ✅ | Cần bot với Message Content Intent |
| WhatsApp | Beta ⚠️ | Risk ban tài khoản |
| Signal | Stable ✅ | Cần signal-cli |
| iMessage | macOS only | AppleScript |
| Slack | Stable ✅ | Cần Slack app với permissions |
| Google Chat | Stable ✅ | Cần Google Workspace |
| Teams | Stable ✅ | Cần Azure bot registration |
| Matrix | Plugin | `openclaw plugins install matrix` |
| Mattermost | Plugin | Self-hosted Mattermost |
| IRC | Plugin | Legacy |
| LINE | Plugin | LINE Messaging API |
| Zalo | Plugin | Zalo Official Account |
| Nostr | Plugin | Decentralized social |
| WeChat | Experimental | Giới hạn ở Trung Quốc |
