# Phần 4: Automation System — Cron, Heartbeat, Hooks

---

## 4.1 Decision matrix — dùng cái nào?

| Bạn cần gì | Cơ chế | Lý do |
|-----------|--------|-------|
| Chính xác giờ giấc (7:00 AM hàng ngày) | **Cron** | Precise scheduling, có lịch sử run, isolated session |
| Kiểm tra định kỳ, alert khi có vấn đề | **Heartbeat** | Lightweight, không tạo task records, tự suppress khi không có gì |
| Phản ứng ngay khi event xảy ra | **Hook** | Event-driven, zero cost khi không có event |
| Nhiều bước phức tạp, có dependency | **Task Flow** | Orchestrate nhiều agent/tool theo DAG |

---

## 4.2 Cron Jobs — đầy đủ

### Syntax tạo cron job

```bash
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 7 * * 1-5" \       # Cron expression: T2-T6 lúc 7:00
  --tz "Asia/Ho_Chi_Minh" \    # QUAN TRỌNG nếu VPS ở timezone UTC
  --session isolated \          # isolated (mặc định) hoặc main
  --announce \                  # Gửi kết quả về channel
  --channel telegram \          # Kênh nhận kết quả
  --model "openrouter/google/gemini-2.5-pro" \  # Override model
  --message "Tóm tắt emails quan trọng 24h qua, thời tiết Hà Nội, và 3 việc cần làm nhất hôm nay."
```

### Ba loại schedule

```bash
# 1. Cron expression (schedule lặp lại)
openclaw cron add \
  --name "weekly-report" \
  --cron "0 9 * * 1" \          # Thứ 2 hàng tuần lúc 9:00
  --tz "Asia/Ho_Chi_Minh" \
  --session isolated \
  --announce \
  --message "Tóm tắt hoạt động tuần qua và mục tiêu tuần tới."

# 2. One-shot (chạy một lần)
openclaw cron add \
  --name "deploy-reminder" \
  --at "2026-05-01T14:00:00+07:00" \  # ISO 8601 với timezone
  --session isolated \
  --announce \
  --message "Nhắc: Deployment lên production đã được lên lịch bây giờ. Confirm proceed?"

# 3. Interval (cách đều nhau)
openclaw cron add \
  --name "server-monitor" \
  --every "2h" \
  --session isolated \
  --message "Fetch https://api.myapp.com/health và alert nếu status không phải 200."
```

### Cron expressions thường dùng

| Expression | Ý nghĩa |
|-----------|---------|
| `0 7 * * 1-5` | T2-T6 lúc 7:00 AM |
| `0 7 * * *` | Mỗi ngày lúc 7:00 AM |
| `0 9 * * 1` | Thứ 2 hàng tuần lúc 9:00 AM |
| `*/30 * * * *` | Mỗi 30 phút |
| `0 0 1 * *` | Đầu tháng lúc 00:00 |
| `0 2 * * 0` | Chủ nhật lúc 2:00 AM (backup) |
| `0 8,12,17 * * 1-5` | T2-T6 lúc 8h, 12h, 17h |

### Management commands

```bash
openclaw cron list                       # Danh sách tất cả jobs + status
openclaw cron show <job-id>              # Chi tiết 1 job
openclaw cron run <job-id>               # Force run ngay lập tức (test)
openclaw cron runs --id <job-id>         # Xem lịch sử runs + logs
openclaw cron disable <job-id>           # Tạm dừng (giữ lại job)
openclaw cron enable <job-id>            # Kích hoạt lại
openclaw cron edit <job-id> --cron "0 8 * * *"   # Đổi schedule
openclaw cron edit <job-id> --model "claude-opus-4-6"  # Đổi model
openclaw cron rm <job-id>                # Xóa vĩnh viễn
openclaw cron status                     # Sức khỏe scheduler
```

### Session modes — so sánh chi tiết

| Mode | Context agent nhận được | Token cost | Dùng khi |
|------|------------------------|-----------|---------|
| `isolated` (mặc định, **khuyến nghị**) | Chỉ workspace files (SOUL, AGENTS...) + job prompt | ~2-5K tokens | Hầu hết: briefing, backup, monitoring, scraping |
| `main` | Toàn bộ chat history + workspace | ~100K tokens | Job cần biết context cuộc hội thoại gần đây |

> 💡 **Luôn dùng `isolated`** trừ khi có lý do cụ thể cần `main`. Tiết kiệm 95% chi phí token.

> ⚠️ **Timezone trap**: Nếu deploy Gateway lên VPS ở UTC (AWS, GCP, DigitalOcean mặc định), cron `0 7 * * *` sẽ chạy lúc 7:00 UTC = 14:00 giờ Việt Nam. Luôn thêm `--tz "Asia/Ho_Chi_Minh"`.

### Ví dụ thực tế — bộ cron hoàn chỉnh cho trợ lý cá nhân

```bash
# Briefing sáng (T2-T6)
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 7 * * 1-5" \
  --tz "Asia/Ho_Chi_Minh" \
  --session isolated \
  --announce \
  --channel telegram \
  --model "openrouter/google/gemini-2.5-pro" \
  --message "Xem lịch hôm nay trong Google Calendar. Tóm tắt email mới từ tối qua. Nhắc các task chưa hoàn thành từ hôm qua. Gộp thành briefing ngắn gọn."

# Tóm tắt cuối ngày
openclaw cron add \
  --name "evening-summary" \
  --cron "0 18 * * 1-5" \
  --tz "Asia/Ho_Chi_Minh" \
  --session isolated \
  --announce \
  --channel telegram \
  --message "Tóm tắt những gì đã làm hôm nay. Liệt kê việc chưa xong. Ghi vào MEMORY.md."

# Backup workspace (Chủ nhật 2:00 AM)
openclaw cron add \
  --name "weekly-backup" \
  --cron "0 2 * * 0" \
  --tz "Asia/Ho_Chi_Minh" \
  --session isolated \
  --message "Chạy openclaw backup create. Xác nhận thành công và log kết quả vào memory."

# Monitor server (mỗi 2h)
openclaw cron add \
  --name "server-health" \
  --every "2h" \
  --session isolated \
  --model "ollama/llama3.2:latest" \
  --message "Kiểm tra https://myapp.com/health. Alert qua Telegram nếu không phải status 200."
```

---

## 4.3 Heartbeat — cấu hình và HEARTBEAT.md

Heartbeat khác Cron ở điểm: **không tạo background task records**, chạy nhẹ trong main session context, tự suppress khi không có gì cần báo.

### Config trong openclaw.json

```json
{
  "agents": {
    "defaults": {
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
        "ackMaxChars": 300,
        "showOk": false,
        "showAlerts": true
      }
    }
  }
}
```

### Giải thích từng field

| Field | Mặc định | Ý nghĩa |
|-------|---------|---------|
| `every` | `"30m"` | Interval chạy heartbeat |
| `target` | `"none"` | Gửi kết quả về đâu: `"last"` (channel gần nhất), `"none"` (silent) |
| `lightContext` | `false` | Chỉ inject HEARTBEAT.md, bỏ các file workspace khác — tiết kiệm token |
| `isolatedSession` | `false` | Fresh session mỗi lần — giảm token từ 100K → 2-5K |
| `activeHours` | không giới hạn | Chỉ chạy trong khung giờ này |
| `ackMaxChars` | `300` | Ký tự tối đa sau HEARTBEAT_OK trước khi suppress |
| `showOk` | `true` | Có hiển thị HEARTBEAT_OK không |
| `showAlerts` | `true` | Có deliver alert messages không |

### CLI config nhanh

```bash
openclaw config set agents.defaults.heartbeat.enabled true
openclaw config set agents.defaults.heartbeat.every "30m"
openclaw config set agents.defaults.heartbeat.model "openrouter/google/gemini-2.5-flash"
openclaw config set agents.defaults.heartbeat.target "last"
openclaw config set agents.defaults.heartbeat.lightContext true
openclaw config set agents.defaults.heartbeat.isolatedSession true
```

### HEARTBEAT.md — template và best practices

```markdown
## Tác vụ kiểm tra định kỳ

tasks:
- name: inbox-scan
  interval: 30m
  prompt: "Kiểm tra inbox Gmail. Nếu có email từ sếp hoặc client cần reply gấp, alert ngay.
           Nếu không có gì urgent, reply HEARTBEAT_OK."

- name: server-monitor
  interval: 1h
  prompt: "Fetch https://api.myapp.com/health. Alert nếu status không phải 200 hoặc
           response > 2000ms. Nếu OK, reply HEARTBEAT_OK."

- name: daily-review
  interval: 24h
  prompt: "Tóm tắt ngày hôm nay: task đã làm, task pending, ghi vào MEMORY.md.
           Gửi summary qua Telegram."
```

> 💡 **Giữ HEARTBEAT.md dưới 50 dòng**. Mỗi lần chạy, toàn bộ file bị consume làm token. 500 dòng × mỗi 30 phút = tốn kém.

### Response contract

```
Agent PHẢI reply HEARTBEAT_OK khi không có gì cần báo.
Gateway tự suppress message nếu chỉ có HEARTBEAT_OK (không spam user).

Nếu có alert → reply bình thường, KHÔNG kèm HEARTBEAT_OK.
```

### Trigger heartbeat thủ công

```bash
openclaw system event --text "Check everything now" --mode now
```

### Precedence của heartbeat config

Config heartbeat được merge theo thứ tự (sau override trước):
1. `agents.defaults.heartbeat` — global
2. `agents.list[agent].heartbeat` — per-agent override
3. `channels.defaults.heartbeat` — channel-level
4. `channels.<name>.heartbeat` — per-channel override

---

## 4.4 Hooks — event-driven automation

Hooks chạy khi có **event** xảy ra, không theo lịch. Zero cost khi không có event.

```bash
# Xem tất cả hooks có sẵn
openclaw hooks list

# Bật hook
openclaw hooks enable /new       # Chạy khi session mới bắt đầu
openclaw hooks enable /reset     # Chạy khi session bị reset

# Kiểm tra
openclaw hooks status
```

### Events phổ biến

| Event | Trigger khi | Dùng để |
|-------|-------------|---------|
| `/new` | Session mới tạo | Auto-load project context, greeting |
| `/reset` | User dùng /reset command | Clear state, reinitialize |
| Gateway boot | Gateway start/restart | Health check, send notification |
| Session compaction | Context bị compress | Backup important context sang MEMORY.md |

### Ví dụ hook trong AGENTS.md

```markdown
## Khi session mới (hook: /new)
1. Đọc USER.md để biết timezone và preferences của user.
2. Đọc MEMORY.md để biết context gần đây.
3. Kiểm tra có task pending nào từ hôm qua không.
4. Nếu là sáng sớm (trước 9h), hỏi user muốn briefing không.
```

---

## 4.5 Task Flow — multi-step automation

Dùng cho workflows phức tạp có nhiều bước và dependency:

```bash
# Tạo task flow từ file
openclaw task create --file workflow.json

# Hoặc inline
openclaw agent \
  --to "my-session" \
  --message "Tạo task flow: 1) Fetch dữ liệu từ API, 2) Phân tích và tóm tắt, 3) Gửi report qua email"
```

Task Flow phù hợp khi: có nhiều agent xử lý song song, có điều kiện rẽ nhánh, cần retry logic, hoặc output của step này là input của step sau.
