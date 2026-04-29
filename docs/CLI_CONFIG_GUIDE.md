# Hướng Dẫn Sử Dụng OpenClaw CLI Để Cấu Hình

Thay vì chỉnh sửa trực tiếp file `openclaw.json` (rất dễ bị sai format hoặc bị hệ thống tự động lưu đè - clobber), OpenClaw cung cấp bộ công cụ CLI mạnh mẽ để cấu hình an toàn ngay từ Terminal. 

Dưới đây là các lệnh CLI quan trọng nhất mà bạn nên biết.

## 1. Nhóm Lệnh `openclaw config`
Sử dụng để can thiệp trực tiếp vào file cấu hình `openclaw.json` một cách chuẩn xác theo schema.

- **Lấy giá trị hiện tại:**
  ```bash
  openclaw config get <đường_dẫn_key>
  # Ví dụ: openclaw config get channels.telegram.enabled
  ```

- **Thay đổi / Thêm mới cấu hình:**
  ```bash
  openclaw config set <đường_dẫn_key> <giá_trị>
  # Ví dụ (chuỗi): openclaw config set gateway.port 19001
  
  # Ví dụ (thêm mảng JSON, cần flag --strict-json):
  openclaw config set commands.ownerAllowFrom '["721480982"]' --strict-json
  ```

- **Xoá một cấu hình (trở về mặc định):**
  ```bash
  openclaw config unset <đường_dẫn_key>
  # Ví dụ: openclaw config unset commands.ownerAllowFrom
  ```

- **Kiểm tra file cấu hình có hợp lệ không:**
  ```bash
  openclaw config validate
  # Lệnh này sẽ kiểm tra openclaw.json với Schema mà không cần khởi động Gateway
  ```

- **Hiển thị đường dẫn file cấu hình đang dùng:**
  ```bash
  openclaw config file
  ```

- **Xem toàn bộ Schema (JSON format):**
  ```bash
  openclaw config schema
  ```

## 2. Nhóm Lệnh `openclaw channels`
Sử dụng để quản lý các kênh giao tiếp (Telegram, WhatsApp, Slack, v.v.).

- **Xem danh sách các kênh đang được bật:**
  ```bash
  openclaw channels list
  ```

- **Kiểm tra trạng thái kết nối của các kênh:**
  ```bash
  openclaw channels status
  # Dùng openclaw channels status --probe để bắt buộc check health ngay lập tức.
  ```

- **Thêm một kênh mới (ví dụ Telegram):**
  ```bash
  openclaw channels add --channel telegram --token <YOUR_BOT_TOKEN>
  ```

- **Đăng nhập vào một kênh (như WhatsApp Web):**
  ```bash
  openclaw channels login --channel whatsapp
  # Hệ thống sẽ sinh mã QR ngay trong terminal để bạn quét
  ```

- **Xem Logs của kênh:**
  ```bash
  openclaw channels logs
  ```

## 3. Quản Lý Models (`openclaw models`)
Để cấu hình mô hình LLM chính thức (Primary Model) hoặc quản lý danh sách mô hình.

- **Xem danh sách các model đang cấu hình:**
  ```bash
  openclaw models list
  ```
- **Thay đổi Primary Model cho Agent:**
  Sử dụng `openclaw config set` để đổi model chính trong hệ thống:
  ```bash
  openclaw config set agents.defaults.model.primary "custom-localhost-20128/nvidia/moonshotai/kimi-k2.5"
  ```
- **Cập nhật context window/max tokens cho model:**
  ```bash
  openclaw config set models.providers.custom-localhost-20128.models.0.contextWindow 128000
  ```

## 4. Quản Lý Skills (`openclaw skills`)
Skills là các kỹ năng mở rộng (như Notion, Slack, hệ điều hành).

- **Danh sách các skills khả dụng:**
  ```bash
  openclaw skills list
  ```
- **Xem thông tin chi tiết của một skill:**
  ```bash
  openclaw skills inspect <tên_skill>
  ```
- **Bật / Tắt một skill:**
  ```bash
  openclaw config set skills.entries.<tên_skill>.enabled true
  ```

## 5. Quản Lý Plugins & Browser
Browser là một dạng plugin đặc biệt (Native MCP Tool) được tích hợp sẵn.

- **Bật công cụ Browser (Trình duyệt):**
  ```bash
  openclaw config set plugins.entries.browser.enabled true
  ```
- **Bật / Tắt các Plugin khác (ví dụ: searxng):**
  ```bash
  openclaw config set plugins.entries.searxng.enabled true
  ```
- **Cấu hình đường dẫn Chrome thủ công (nếu cần):**
  ```bash
  openclaw config set browser.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
  ```

## 6. Cấu Hình Quyền & Commands
Quản lý quyền hạn thực thi lệnh (slash commands) hoặc quyền Admin.

- **Cấp quyền Admin (Owner) cho một tài khoản (ví dụ Telegram):**
  ```bash
  openclaw config set commands.ownerAllowFrom '["721480982"]' --strict-json
  ```
- **Bật tính năng Slash Commands (/commands) trên các kênh:**
  ```bash
  openclaw config set commands.native true
  ```
- **Cho phép bot tự khởi động lại qua tin nhắn (/restart):**
  ```bash
  openclaw config set commands.restart true
  ```

## Lời Khuyên
- Mỗi khi dùng lệnh `openclaw config set` thành công, OpenClaw CLI sẽ tự động sao lưu file `openclaw.json` cũ thành file `.bak`. Nếu cần, bạn có thể dễ dàng khôi phục lại.
- **Lưu ý quan trọng:** Một số thay đổi cấu hình sâu (như đổi quyền owner, đổi port, kích hoạt plugin) yêu cầu bạn phải **khởi động lại Gateway** (`Ctrl + C` và chạy lại) thì mới có tác dụng.
