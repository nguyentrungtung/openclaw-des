# Phần 5: Browser Automation — Đầy Đủ

---

## 5.1 Kiến trúc browser trong OpenClaw

```
Agent (LLM)
    ↓ browser tool call (ngôn ngữ tự nhiên)
Gateway (port 18789)
    ↓ dịch sang CDP command
Control Service (port 18791)
    ↓ ChromeDevTools Protocol
Browser Profiles (ports 18800-18899)
    ↓ DOM manipulation
Web Page
```

Extension Relay (port 18792) cho phép kiểm soát Chrome **thật** của bạn (với login sessions).

### Port family (relative với gateway.port = 18789)

| Offset | Port mặc định | Service |
|--------|--------------|---------|
| +0 | 18789 | Gateway |
| +2 | 18791 | Control Service |
| +3 | 18792 | Extension Relay |
| +11 → +110 | 18800 → 18899 | Managed browser profiles |

> Nếu đổi `gateway.port`, tất cả ports trên tự dịch chuyển theo.

---

## 5.2 Ba browser modes — so sánh đầy đủ

| | Managed Profile | Existing-Session Profile | Remote CDP |
|--|----------------|------------------------|------------|
| **Isolation** | Cao — Chromium riêng, data dir sạch | Thấp — share Chrome thật | Trung bình |
| **Login state** | ❌ Fresh mỗi lần | ✅ Giữ cookies/sessions | Tùy cấu hình |
| **Setup** | Đơn giản, out-of-the-box | Cần enable remote debugging trên Chrome | Cần CDP WebSocket URL |
| **Hiển thị** | headful hoặc headless | Chrome thật của bạn | N/A (remote) |
| **Bảo mật** | Cao nhất | Truy cập toàn bộ Chrome profile | Tùy host |
| **Use case chính** | Scraping, form fill, automation sạch | Facebook, Shopee, ngân hàng — cần login | Cloud deploy, Browserless |

---

## 5.3 Setup Managed Mode (khuyến nghị)

```bash
# Start browser (dùng default profile)
openclaw browser start

# Hoặc chỉ định profile
openclaw browser --browser-profile openclaw start

# Kiểm tra status
openclaw browser status

# Test nhanh
openclaw browser open https://example.com
openclaw browser snapshot
openclaw browser screenshot
```

### Config multi-profile

```json
{
  "browser": {
    "enabled": true,
    "headless": false,
    "noSandbox": false,
    "defaultProfile": "openclaw",
    "profiles": {
      "openclaw": {
        "cdpPort": 18800
      },
      "work": {
        "cdpPort": 18801
      },
      "shopping": {
        "cdpPort": 18802
      }
    }
  }
}
```

---

## 5.4 Setup Existing-Session Mode (cho tasks cần login)

### macOS/Linux

Bật remote debugging trên Chrome/Brave:

```bash
# macOS — Chrome
open -a "Google Chrome" --args --remote-debugging-port=9222

# Linux — Chrome
google-chrome --remote-debugging-port=9222

# Hoặc dùng existing-session driver (tự attach)
openclaw config set 'browser.profiles.user.driver' "existing-session"
openclaw config set 'browser.profiles.user.attachOnly' true
openclaw browser --browser-profile user start
```

### WSL2 Windows — phức tạp, cần port proxy

Extension trong Chrome (Windows) cần kết nối tới port 18792 đang chạy trong WSL2.

```powershell
# PowerShell (Run as Administrator)
$wslIP = (wsl hostname -I).Trim()
netsh interface portproxy add v4tov4 `
  listenport=18792 `
  listenaddress=127.0.0.1 `
  connectport=18792 `
  connectaddress=$wslIP

# Xác nhận
netsh interface portproxy show all

# Xóa rule (khi không cần)
netsh interface portproxy delete v4tov4 listenport=18792 listenaddress=127.0.0.1
```

> ⚠️ Rule portproxy **mất sau mỗi lần reboot**. Thêm vào Task Scheduler để tự chạy khi startup với quyền Administrator.

### Remote CDP (cloud/Browserless)

```json
{
  "browser": {
    "profiles": {
      "cloud": {
        "cdpUrl": "wss://production-sfo.browserless.io?token=YOUR_TOKEN"
      }
    }
  }
}
```

```bash
openclaw browser --browser-profile cloud start
openclaw browser --browser-profile cloud open https://example.com
```

---

## 5.5 Browser CLI commands — toàn bộ

### Lifecycle

```bash
openclaw browser start                   # Start browser (default profile)
openclaw browser stop                    # Stop browser
openclaw browser status                  # Kiểm tra trạng thái
openclaw browser restart                 # Restart browser
```

### Navigation

```bash
openclaw browser open https://example.com       # Mở tab mới
openclaw browser navigate https://example.com   # Navigate tab hiện tại
openclaw browser tabs                           # List tất cả tabs
openclaw browser tab select 2                   # Chuyển sang tab #2
openclaw browser close <targetId>               # Đóng tab
```

### Page Reading

```bash
openclaw browser snapshot                        # AI snapshot (cần Playwright)
openclaw browser snapshot --format aria          # Accessibility tree format
openclaw browser snapshot --interactive          # Role-based refs (ổn định hơn)
openclaw browser snapshot --labels               # Ảnh + refs overlay (debug)
openclaw browser screenshot                      # Chụp viewport hiện tại
openclaw browser screenshot --full-page          # Chụp toàn bộ trang
openclaw browser screenshot --ref e5             # Chụp element cụ thể
openclaw browser pdf                             # Xuất PDF (cần Playwright)
```

### Interaction

```bash
openclaw browser click <ref>                     # Click element
openclaw browser click <ref> --double           # Double click
openclaw browser type <ref> "text"              # Nhập text vào field
openclaw browser type <ref> "text" --submit     # Nhập text + Enter
openclaw browser fill --fields '[{"ref":"e1","value":"John"},{"ref":"e2","value":"Doe"}]'
openclaw browser hover <ref>                    # Hover
openclaw browser drag <ref1> <ref2>             # Drag and drop
openclaw browser select <ref> "Option A"        # Dropdown select
openclaw browser press Enter                    # Nhấn phím
openclaw browser press "Control+A"              # Key combo
openclaw browser press "Escape"                 # Escape
openclaw browser scrollintoview <ref>           # Scroll đến element
```

### Wait & Verify

```bash
openclaw browser wait --text "Thành công"        # Chờ text xuất hiện trên trang
openclaw browser wait --url "**/dashboard"       # Chờ URL match pattern
openclaw browser wait --load networkidle         # Chờ network idle
openclaw browser wait --fn "window.appLoaded"   # Chờ JS condition
openclaw browser wait --ms 2000                  # Chờ milliseconds
```

### Forms & Files

```bash
openclaw browser upload /path/to/file.pdf        # Upload file
openclaw browser dialog --accept                 # Accept alert/confirm dialog
openclaw browser dialog --dismiss                # Dismiss dialog
openclaw browser dialog --prompt "typed text"    # Điền vào prompt dialog
```

### Data Extraction

```bash
openclaw browser cookies                          # Xem tất cả cookies
openclaw browser cookies set name value --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage                          # localStorage + sessionStorage
openclaw browser storage local get key
openclaw browser storage local set key value
openclaw browser evaluate --fn '(el) => el.textContent' --ref e5  # JS trên element
openclaw browser requests --filter api            # Network requests đã ghi
openclaw browser errors                           # Page errors
openclaw browser errors --clear
openclaw browser console --level error            # Console logs theo level
```

### Browser State

```bash
openclaw browser set offline on                   # Offline mode
openclaw browser set offline off
openclaw browser set headers --headers-json '{"Accept-Language":"vi-VN","X-Debug":"1"}'
openclaw browser set timezone "Asia/Ho_Chi_Minh"
openclaw browser set geo 21.028511 105.804817 --origin "https://maps.google.com"
openclaw browser set device "iPhone 14"           # Device emulation
openclaw browser set device "iPad Pro"
openclaw browser set media dark                   # Dark mode
openclaw browser set media light
```

### Profile Management

```bash
openclaw browser profiles                         # List tất cả profiles
openclaw browser --browser-profile work start     # Start profile cụ thể
openclaw browser create-profile --name headless --driver openclaw
openclaw browser reset-profile --name openclaw    # Xóa cookies/data (giữ profile)
openclaw browser delete-profile --name temp       # Xóa hoàn toàn profile
```

### Debug

```bash
openclaw browser trace start                      # Bắt đầu record trace
openclaw browser trace stop                       # Dừng và lưu trace
openclaw browser highlight <ref>                  # Highlight element (visual debug)
```

---

## 5.6 Snapshot refs — cách hoạt động

Snapshot trả về accessibility tree, mỗi interactive element có `ref`:

```
[AI Snapshot output]
Page: CGV Viet Nam - Mua Ve Online
Navigation:
  Link "Trang chủ" [ref=e1]
  Link "Phim đang chiếu" [ref=e2]
  Button "Tìm kiếm" [ref=e3]
Main content:
  Heading "Oppenheimer" [ref=e10]
  Button "Mua vé" [ref=e11]
  ...
```

**Quy tắc sử dụng refs:**

> ⚠️ **Refs expire sau mỗi navigation**. Sau khi click gây chuyển trang, phải `snapshot` lại để lấy refs mới.

> 💡 Dùng `--interactive` (role-based refs dạng `e1`, `e2`) thay vì AI snapshot khi structure phức tạp — ổn định hơn với dynamic content.

> 💡 Dùng `--labels` khi debug thủ công — trả về ảnh chụp với số hiệu overlay lên các element.

---

## 5.7 Workflow thực tế — Điền form đặt vé CGV

```bash
# Bước 1: Mở trang, chờ load
openclaw browser open "https://cgv.vn"
openclaw browser wait --load networkidle
openclaw browser snapshot

# Bước 2: Tìm và click "Mua vé"
# (refs từ snapshot bước 1)
openclaw browser click e11
openclaw browser wait --load networkidle
openclaw browser snapshot    # BẮT BUỘC snapshot lại sau navigation

# Bước 3: Chọn phim
openclaw browser click e23   # Link tên phim
openclaw browser wait --text "Chọn suất chiếu"
openclaw browser snapshot

# Bước 4: Chọn suất chiếu
openclaw browser click e45   # "20:00 - Rạp 3 - IMAX"
openclaw browser wait --load networkidle
openclaw browser snapshot

# Bước 5: Chọn ghế
openclaw browser click e67   # Ghế A5

# ⚠️ Bước 6: DỪNG — xác nhận với user trước khi thanh toán
openclaw browser screenshot --path /tmp/cgv-seats.png
# → Gửi screenshot cho user: "Bạn đã chọn ghế A5, suất 20:00 Rạp 3. Xác nhận thanh toán?"

# Bước 7: Sau khi user xác nhận
openclaw browser click e89   # "Tiến hành thanh toán"
openclaw browser wait --url "**/payment"
openclaw browser snapshot
```

### Workflow tổng quát điền form

```bash
# 1. Navigate
openclaw browser open <url>
openclaw browser wait --load networkidle

# 2. Snapshot để lấy refs
openclaw browser snapshot --interactive

# 3. Điền form nhiều field
openclaw browser fill --fields '[
  {"ref":"e5","value":"Nguyen Van A"},
  {"ref":"e7","value":"nguyenvana@email.com"},
  {"ref":"e9","value":"0912345678"}
]'

# 4. Submit
openclaw browser click e15   # Submit button
openclaw browser wait --text "Thành công"

# 5. Screenshot kết quả
openclaw browser screenshot
```

---

## 5.8 Cài Playwright (bắt buộc cho snapshot/PDF) {#playwright-install}

Playwright **không** được bundle cùng openclaw package, phải cài riêng.

### macOS / Linux

```bash
# Tìm thư mục openclaw
cd $(npm root -g)/openclaw

# Cài playwright
npm install playwright --legacy-peer-deps

# Download Chromium browser
npx playwright install chromium

# Restart gateway
openclaw gateway restart

# Verify
openclaw doctor --fix
openclaw browser open https://example.com
openclaw browser snapshot   # Phải chạy được
```

### Docker

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium

# Persist downloads qua volume mount
# Đặt PLAYWRIGHT_BROWSERS_PATH trong docker-compose.yml
```

### Verify Playwright hoạt động

```bash
openclaw doctor --fix
# → Nếu output có "playwright: ok" là thành công

# Hoặc test trực tiếp
openclaw browser start
openclaw browser open https://example.com
openclaw browser snapshot
# Nếu snapshot trả về text thay vì lỗi 501 → OK
```

> ⚠️ Nếu `snapshot` trả về `HTTP 501`, Playwright chưa cài đúng. Chạy `openclaw doctor --fix` để tự sửa.

---

## 5.9 SSRF Security Policy

Mặc định, browser tool **chặn** navigation tới private network (localhost, 192.168.x.x...).

```json
{
  "browser": {
    "ssrfPolicy": {
      "dangerouslyAllowPrivateNetwork": false,
      "hostnameAllowlist": ["*.mycompany.com"],
      "allowedHostnames": ["api.internal"]
    }
  }
}
```

> ⚠️ Chỉ set `dangerouslyAllowPrivateNetwork: true` trong môi trường dev hoàn toàn trusted. Không để như vậy trên production.
