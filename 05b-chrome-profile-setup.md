# Chrome Hệ Thống + Profile Riêng cho OpenClaw

> Thiết lập này cho phép OpenClaw điều khiển Chrome thật của máy (không phải Chromium stripped-down),
> dùng profile độc lập — không đụng Chrome hằng ngày của bạn.
> Chrome chạy **headful** (cửa sổ thật, user nhìn thấy trực tiếp) với mức độ chống bot detection cao nhất.

---

## Tại sao cần cách này

| | Playwright Chromium | Chrome for Testing | **Chrome hệ thống + profile riêng** |
|--|--------------------|--------------------|--------------------------------------|
| Bot detection | Dễ bị phát hiện | Khó hơn | Khó nhất |
| Headful (user nhìn thấy) | Tùy config | Tùy config | **Luôn luôn** |
| Cookies/login sẵn | Không | Không | Có (tự đăng nhập 1 lần) |
| Fingerprint | Stripped-down | Gần Chrome thật | Chrome thật 100% |
| `navigator.webdriver` | Bị lộ | Bị lộ | Ẩn (với flag đúng) |
| Đụng Chrome hằng ngày | Không | Không | Không (profile riêng) |

Phù hợp cho: đặt vé phim, đặt vé máy bay, đăng bài Facebook, điền form, scraping trang có đăng nhập.

> **Headful vs headless:** Với driver `existing-session`, Chrome LUÔN chạy headful — cửa sổ Chrome
> hiện ra thật, bạn nhìn thấy agent đang làm gì theo thời gian thực. Không cần config gì thêm.

---

## Yêu cầu

- macOS với Chrome đã cài (kiểm tra: `ls /Applications/Google\ Chrome.app`)
- OpenClaw đã cài qua local prefix installer (`~/.openclaw/`)
- Playwright đã cài vào Node của openclaw (xem [02-installation.md](02-installation.md))

---

## Bước 1 — Tạo thư mục profile riêng

```bash
mkdir -p ~/.openclaw/chrome-profile
```

Profile này hoàn toàn tách biệt với Chrome hằng ngày của bạn (`~/Library/Application Support/Google/Chrome/`).

---

## Bước 2 — Mở Chrome với profile riêng, đăng nhập lần đầu

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --user-data-dir="$HOME/.openclaw/chrome-profile" \
  --no-first-run \
  --no-default-browser-check
```

Cửa sổ Chrome mới sẽ bật lên (profile trống, không có lịch sử).

**Đăng nhập thủ công tất cả tài khoản cần dùng:**
- Facebook / Instagram
- CGV, Galaxy Cinema, BHD...
- Vé máy bay (VietJet, Bamboo, Vietnam Airlines)
- Bất kỳ trang nào bạn muốn openclaw thao tác

Sau khi đăng nhập xong → đóng Chrome lại. Cookies đã lưu vào `~/.openclaw/chrome-profile/`.

> ⚠️ **Không copy profile Chrome cá nhân** — Chrome gắn session token với profile gốc,
> copy sang profile khác sẽ bị logout hoặc Chrome từ chối load.

---

## Bước 3 — Cài playwright-extra và stealth plugin

```bash
~/.openclaw/tools/node/bin/npm install -g \
  playwright-extra \
  puppeteer-extra-plugin-stealth
```

`playwright-extra` + `stealth` vá các điểm Playwright bị lộ với website:
- `navigator.webdriver` flag
- Chrome runtime object
- Plugin/mimeType lists
- Canvas fingerprint
- WebGL vendor strings

---

## Bước 4 — Cấu hình OpenClaw

### 4a — Trỏ executablePath vào Chrome hệ thống

```bash
openclaw config set browser.executablePath \
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

### 4b — Đăng ký profile trong openclaw.json

> ⚠️ **Không dùng `openclaw browser create-profile`** cho existing-session — lệnh đó chỉ hoạt động với
> driver `openclaw` (managed Chromium). Với Chrome thật, phải khai báo trực tiếp trong `openclaw.json`.

Mở config:

```bash
openclaw config edit
```

Thêm vào block `"browser"`:

```jsonc
{
  "browser": {
    "executablePath": "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
    "defaultProfile": "personal",
    "profiles": {
      "personal": {
        "driver": "existing-session",
        "userDataDir": "/Users/<bạn>/.openclaw/chrome-profile",
        "cdpPort": 9222,
        "color": "#4285F4"
      }
    }
  }
}
```

> ⚠️ **`color` phải là hex string** (ví dụ `#4285F4`) — không dùng tên màu như `"blue"`, gateway sẽ reject
> và revert config về bản cũ.

Restart gateway để áp dụng:

```bash
openclaw gateway restart
```

### 4c — Kiểm tra profile đã đăng ký

```bash
openclaw browser profiles
```

Output mong đợi:

```
personal: stopped [existing-session]
  transport: chrome-mcp, color: #4285F4
openclaw: stopped [default]
  port: 18800, color: #FF4500
```

---

## Bước 5 — Khởi động Chrome và kết nối OpenClaw

Driver `existing-session` dùng transport `chrome-mcp` — OpenClaw **không tự mở Chrome**, bạn phải chạy
Chrome với `--remote-debugging-port` trước, rồi openclaw mới attach vào.

### 5a — Tạo startup script (làm một lần)

```bash
cat > ~/.openclaw/start-browser.sh << 'EOF'
#!/bin/bash
# Start Chrome với remote debugging + anti-bot cho openclaw personal profile

# Kill Chrome cũ nếu có
pkill -f "Google Chrome" 2>/dev/null
sleep 2

# Mở Chrome với remote debugging + anti-bot flags
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --user-data-dir="$HOME/.openclaw/chrome-profile" \
  --remote-debugging-port=9222 \
  --no-first-run \
  --no-default-browser-check \
  --disable-blink-features=AutomationControlled \
  2>/dev/null &

# Chờ Chrome khởi động
echo "Waiting for Chrome..."
for i in $(seq 1 15); do
  sleep 1
  if curl -s http://localhost:9222/json/version > /dev/null 2>&1; then
    break
  fi
done

# Tạo DevToolsActivePort (Chrome 147+ không tự tạo file này)
BROWSER_WS=$(curl -s http://localhost:9222/json/version \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['webSocketDebuggerUrl'])" \
  | sed 's|ws://localhost:9222||')
printf "9222\n%s" "$BROWSER_WS" > "$HOME/.openclaw/chrome-profile/DevToolsActivePort"

# Connect openclaw vào Chrome
~/.openclaw/bin/openclaw browser --browser-profile personal start
EOF
chmod +x ~/.openclaw/start-browser.sh
```

**Giải thích các flag:**

| Flag | Tác dụng |
|------|----------|
| `--remote-debugging-port=9222` | Bật CDP để OpenClaw attach vào |
| `--disable-blink-features=AutomationControlled` | Ẩn dấu hiệu CDP/DevTools với website — tránh bị detect là bot |
| `--no-first-run` | Bỏ qua màn hình welcome của Chrome |

> **Tại sao cần tạo `DevToolsActivePort` thủ công?**
> Chrome 147+ không còn tự ghi file này vào `userDataDir` khi khởi động. OpenClaw đọc file đó để biết
> port kết nối. Script trên lấy port từ CDP API và tạo file đúng format.

### 5b — Tạo alias để gõ nhanh (làm một lần)

```bash
echo 'alias ocbrowser="~/.openclaw/start-browser.sh"' >> ~/.zshrc
source ~/.zshrc
```

### 5c — Chạy mỗi khi muốn dùng

```bash
ocbrowser
```

Output khi thành công:

```
Waiting for Chrome...
🦞 browser [personal] running: true
```

Chrome sẽ bật lên cửa sổ thật — bạn nhìn thấy mọi thao tác agent thực hiện theo thời gian thực.

### 5d — Kiểm tra

```bash
# Xem trạng thái
openclaw browser --browser-profile personal status

# Thử mở một trang
openclaw browser --browser-profile personal open https://facebook.com
```

> ⚠️ **Syntax đúng:** `--browser-profile` là option của `browser`, phải đặt **trước** subcommand:
> ```bash
> # Đúng:
> openclaw browser --browser-profile personal start
>
> # Sai (lỗi "unknown option"):
> openclaw browser start --browser-profile personal
> ```

---

## Anti-bot detection — cơ chế hoạt động

Thiết lập này chống bot theo nhiều lớp:

| Lớp | Cơ chế | Trạng thái |
|-----|--------|-----------|
| Binary thật | Chrome hệ thống, không phải Chromium stripped | ✓ |
| Headful | Cửa sổ thật, có GPU rendering, đầy đủ Chrome internals | ✓ |
| Fingerprint | User-agent, resolution, timezone, fonts của Chrome thật | ✓ |
| Cookies/history | Profile có lịch sử duyệt web thật | ✓ |
| `navigator.webdriver` | Ẩn nhờ `--disable-blink-features=AutomationControlled` | ✓ |
| Tốc độ gõ | Agent dùng `--slowly` flag khi type, bắt chước người thật | ✓ (cấu hình trong TOOLS.md) |
| Tốc độ thao tác | Agent chờ giữa các bước với `browser wait --ms` | ✓ (cấu hình trong TOOLS.md) |

**Những gì agent phải làm để tránh bị detect:**
- Luôn dùng `browser type <ref> "text" --slowly` khi gõ text
- Dùng `browser wait --ms 800` giữa các thao tác lớn
- Không click liên tục nhanh — bắt chước hành vi người thật

---

## Sử dụng hằng ngày

**Mỗi khi bật máy:**

```bash
ocbrowser
```

**Khi openclaw đang điều khiển Chrome:**
- Bạn thấy Chrome thao tác — đây là bình thường, không phải headless
- Không click vào cửa sổ Chrome đó — sẽ làm hỏng automation giữa chừng
- Chrome hằng ngày của bạn vẫn dùng bình thường song song

**Nếu bị logout khỏi tài khoản**, đóng Chrome, mở lại bằng lệnh bên dưới để đăng nhập thủ công, rồi
đóng lại và chạy `ocbrowser`:

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --user-data-dir="$HOME/.openclaw/chrome-profile" \
  --no-first-run \
  --no-default-browser-check
```

**Backup profile sau khi đã setup đăng nhập xong** (tránh phải làm lại):

```bash
cp -r ~/.openclaw/chrome-profile ~/.openclaw/chrome-profile-backup-$(date +%Y%m%d)
```

---

## Model phù hợp cho browser automation

Browser tasks (snapshot → click → type → wait) tốn nhiều token và cần nhiều lượt tool call liên tiếp.
Model nhỏ (≤31B) thường timeout hoặc lặp vòng. Khuyến nghị:

| Model | Tốc độ | Chất lượng | Ghi chú |
|-------|--------|------------|---------|
| `google/gemma-4-31b-it` (Nvidia) | Chậm | Thấp | Default hiện tại — dễ timeout |
| `claude-haiku-4-5-20251001` | Nhanh | Tốt | Rẻ, phù hợp task đơn giản |
| `claude-sonnet-4-6` | Trung bình | Rất tốt | Khuyến nghị cho browser tasks phức tạp |

Đổi model:

```bash
openclaw config set agents.defaults.model.primary "claude-sonnet-4-6"
openclaw gateway restart
```

Nếu muốn giữ model nhỏ nhưng tránh timeout, tắt idle timeout:

```bash
openclaw config set agents.defaults.llm.idleTimeoutSeconds 0
openclaw gateway restart
```

---

## Giới hạn — những gì vẫn có thể bị detect

| Trang | Mức độ | Ghi chú |
|-------|--------|---------|
| CGV, Galaxy, BHD | Thấp | Thường qua được |
| VietJet, Bamboo | Trung bình | Đôi khi có CAPTCHA — openclaw dừng để bạn giải tay |
| Facebook (đăng bài) | Cao | Dùng đúng cookie session + tốc độ gõ tự nhiên là quan trọng |
| Shopee, Tiki | Trung bình | Bot detection trên checkout, ổn trên browsing |

**Facebook cụ thể:** Dùng tài khoản đã có lịch sử hoạt động thật, không dùng tài khoản mới tạo.

---

## Lỗi thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|---------|
| `scope upgrade pending approval` | CLI dùng lại keypair device cũ và xin thêm scope | Mở `openclaw dashboard` → Devices → approve pending request |
| `browser.request cannot mutate persistent browser profiles` | Dùng `create-profile` cho existing-session driver | Khai báo profile trực tiếp trong `openclaw.json` (xem Bước 4b) |
| Config bị revert về bản cũ sau khi edit | Gateway validator reject field không hợp lệ (ví dụ `color: "blue"`) | Dùng hex color như `#4285F4` |
| `error: unknown option '--browser-profile'` | Flag đặt sau subcommand thay vì trước | Đổi thành `openclaw browser --browser-profile personal start` |
| `Could not connect to Chrome` / `ENOENT: DevToolsActivePort` | Chrome 147+ không tạo file `DevToolsActivePort` | Dùng `start-browser.sh` — script tự tạo file sau khi Chrome khởi động |
| `Failed to launch chrome` | Path sai hoặc Chrome đang bị lock bởi session khác | Đóng hết Chrome (`pkill -f "Google Chrome"`), thử lại |
| `Session expired` trên Facebook | Cookie hết hạn | Mở Chrome với profile openclaw, đăng nhập lại |
| CAPTCHA xuất hiện | Trang phát hiện automation | openclaw dừng lại để bạn giải tay, sau đó tiếp tục |
| `net::ERR_CONNECTION_REFUSED` | Gateway chưa chạy | `openclaw gateway start` |
| `The model did not produce a response before the LLM idle timeout` | Model nhỏ (gemma, llama...) xử lý browser snapshot quá chậm, bị timeout | Tắt timeout: `openclaw config set agents.defaults.llm.idleTimeoutSeconds 0` → `openclaw gateway restart`. Hoặc đổi sang model mạnh hơn (xem bên dưới) |
