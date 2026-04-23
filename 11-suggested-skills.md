# Phần 11: Suggested Skills — Trợ Lý Thông Minh Cho Cuộc Sống Hàng Ngày

> Dựa trên dữ liệu cộng đồng ClawHub (3,286 skills, 1.5M+ total downloads, tháng 4/2026).
> Sau đợt **ClawHavoc cleanup** loại bỏ 2,419 skills đáng ngờ — danh sách này chỉ gồm skills
> đã được kiểm chứng: download cao, star rating tốt, security scan "Benign".

---

## Cách cài skill

```bash
# Cú pháp chuẩn
openclaw skills install <slug>

# Xem thông tin trước khi cài (kiểm tra permissions)
openclaw skills info <slug>

# Xem đã cài gì
openclaw skills list

# Update tất cả
openclaw skills update --all
```

---

## NHÓM 1 — Nền Tảng (cài trước tiên)

Những skills này không làm một việc cụ thể — chúng làm agent **thông minh hơn, tự cải thiện, chủ động hơn** theo thời gian. Cài trước khi cài bất cứ thứ gì khác.

---

### 1.1 `capability-evolver` — Agent tự tiến hóa
**#1 ClawHub · 35,581 downloads · ★33**

```bash
openclaw skills install capability-evolver
```

Agent tự phân tích workflow của mình, phát hiện điểm yếu, và **tự đề xuất cải thiện** AGENTS.md, SOUL.md. Theo thời gian agent biết bạn làm việc như thế nào và tối ưu luồng xử lý phù hợp hơn.

Yêu cầu: Không có. Tự chạy định kỳ trong background.

---

### 1.2 `self-improving-agent` — Học từ lỗi và correction
**#4 ClawHub · 15,962 downloads · ★132 (cao nhất toàn ClawHub)**

```bash
openclaw skills install self-improving-agent
```

Mỗi khi bạn sửa agent, agent fail, hoặc bạn nói "không phải như vậy" — skill này capture lại bài học đó vào `.learnings` file. Lần sau agent **không lặp lại lỗi cũ**. Hook tự động trigger sau mỗi correction.

Yêu cầu: Không có. Hoạt động passively qua hooks.

---

### 1.3 `proactive-agent` — Chủ động, không chỉ phản ứng
**#17 ClawHub · 7,010 downloads**

```bash
openclaw skills install proactive-agent
```

Kết hợp: self-reflection + self-criticism + self-learning + self-organizing memory. Agent không chỉ trả lời câu hỏi mà còn **tự đặt câu hỏi**, nhận ra khi nào cần check-in với bạn, và organize memory có hệ thống hơn.

Yêu cầu: Không có. Tích hợp với Heartbeat để chạy định kỳ.

---

## NHÓM 2 — Productivity & Workspace

---

### 2.1 `gog` — Google Workspace đầy đủ
**#6 ClawHub · 14,313 downloads · ★48**

```bash
openclaw skills install gog
```

Tích hợp toàn bộ Google Workspace qua một skill duy nhất:

| Service | Làm được gì |
|---------|-------------|
| **Gmail** | Đọc, soạn, gửi, label, tìm kiếm email |
| **Calendar** | Xem lịch, tạo sự kiện, mời người, check availability |
| **Drive** | Upload, download, search, share file |
| **Docs** | Tạo, chỉnh sửa, export document |
| **Sheets** | Đọc/ghi spreadsheet, formula, pivot |
| **Contacts** | Tra cứu thông tin liên lạc |

Yêu cầu: Google OAuth — agent hướng dẫn login khi cài xong.

Ví dụ dùng:
```
"Kiểm tra Gmail xem có email nào từ client chưa đọc không"
"Tạo event họp team sáng thứ 2, mời everyone@company.com"
"Lưu file này lên Drive vào folder Projects/2026"
```

---

### 2.2 `notion` — Quản lý knowledge base và project
**Top Productivity · Stable**

```bash
openclaw skills install notion
```

Kết nối agent với Notion workspace: tạo page, update database, query project board, tìm kiếm note.

Yêu cầu: `NOTION_API_KEY` — lấy tại notion.so/my-integrations.

Ví dụ dùng:
```
"Tạo note trong Notion với title 'Meeting 23/4' và nội dung sau..."
"Tìm tất cả task trong database Projects có status = In Progress"
"Update trạng thái task #123 thành Done"
```

---

### 2.3 `summarize` — Tóm tắt thông minh mọi thứ
**#8 ClawHub · 10,956 downloads**

```bash
openclaw skills install summarize
```

Tóm tắt bất kỳ nội dung nào với nhiều format khác nhau:
- Executive brief (1 paragraph)
- Bullet points (5-7 điểm chính)
- Q&A format
- TLDR (1 câu)

Input: URL, PDF, đoạn text dài, transcript meeting.

Yêu cầu: Không có.

Ví dụ dùng:
```
"Tóm tắt bài báo này: [URL]"
"Tóm tắt cuộc họp vừa rồi thành action items"
"Đọc PDF này và cho tôi biết 5 điểm quan trọng nhất"
```

---

### 2.4 `mission-control` — Briefing buổi sáng tổng hợp
**Productivity Stack · Recommended**

```bash
openclaw skills install mission-control
```

Tổng hợp mọi thứ cần biết vào đầu ngày: lịch hôm nay, email chưa đọc quan trọng, task pending, tin tức liên quan. Format gọn thành một briefing duy nhất thay vì phải hỏi từng thứ.

Yêu cầu: Cần `gog` cài trước. Tích hợp tốt với Cron job sáng 7h.

```bash
# Thêm cron briefing sáng sau khi cài
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 7 * * 1-5" \
  --tz "Asia/Ho_Chi_Minh" \
  --session isolated \
  --announce \
  --message "Chạy Mission Control briefing cho ngày hôm nay."
```

---

### 2.5 `task-prioritizer` — Sắp xếp ưu tiên task tự động
**Productivity Stack**

```bash
openclaw skills install task-prioritizer
```

Nhận danh sách task → phân tích deadline, dependency, context → trả về thứ tự ưu tiên có giải thích. Dùng framework RICE hoặc MoSCoW tùy context.

Yêu cầu: Không có.

---

### 2.6 `obsidian` — Kết nối Obsidian vault
**#19 ClawHub · 5,791 downloads**

```bash
openclaw skills install obsidian
```

Agent đọc/ghi trực tiếp vào Obsidian vault của bạn. Tạo note, tìm kiếm, link note, append vào daily note.

Yêu cầu: Obsidian đang chạy và vault path được config.

Ví dụ dùng:
```
"Thêm vào daily note hôm nay: meeting lúc 3h với team A"
"Tìm tất cả note có tag #project"
"Tạo note mới 'Research LLM Routing' trong folder Resources"
```

---

## NHÓM 3 — Web & Research

---

### 3.1 `agent-browser` — Điều khiển browser, tự động hóa web
**#7 ClawHub · 11,836 downloads · ★43**

```bash
openclaw skills install agent-browser
```

Dạy agent cách dùng browser tool một cách có hệ thống: navigate, click, fill form, scrape data, chụp screenshot. Đây là skill **wrapper** — cần plugin `browser` và Playwright đã cài (xem Phần 5).

Yêu cầu: `openclaw doctor --fix` phải pass trước khi cài.

Ví dụ dùng:
```
"Vào tiki.vn tìm iPhone 15 Pro, lấy 3 kết quả rẻ nhất"
"Điền form đăng ký hội thảo tại [URL] với thông tin sau..."
"Chụp screenshot trang dashboard của tôi mỗi sáng"
```

---

### 3.2 `tavily-search` — Tìm kiếm web tối ưu cho AI
**#13 ClawHub · 8,142 downloads**

```bash
openclaw skills install tavily-search
```

Thay vì search thông thường trả về danh sách link, Tavily trả về **kết quả đã được xử lý**: relevant snippets, summary, citations. Agent tiêu thụ ít token hơn và kết quả chất lượng hơn.

Yêu cầu: `TAVILY_API_KEY` — đăng ký miễn phí tại tavily.com (1,000 requests/tháng free).

```bash
echo 'TAVILY_API_KEY=tvly-xxxx' >> ~/.openclaw/.env
```

---

### 3.3 `weather` — Thời tiết real-time
**#11 ClawHub · 9,002 downloads**

```bash
openclaw skills install weather
```

Lấy thời tiết hiện tại và dự báo cho bất kỳ địa điểm nào. Hữu ích trong heartbeat hoặc morning briefing.

Yêu cầu: Thường dùng OpenWeatherMap API (free tier đủ dùng) hoặc wttr.in (không cần key).

---

## NHÓM 4 — Developer Tools

---

### 4.1 `github` — Quản lý GitHub từ chat
**#9 ClawHub · 10,611 downloads**

```bash
openclaw skills install github
```

Dùng `gh` CLI để thao tác với GitHub:

| Action | Ví dụ |
|--------|-------|
| PR | Tạo, review, merge, list open PRs |
| Issues | Tạo, close, comment, assign |
| Actions | Xem workflow runs, trigger manually |
| Repos | Clone, fork, search |

Yêu cầu: `gh` CLI đã install và authenticated (`gh auth login`).

Ví dụ dùng:
```
"List tất cả open PR của repo openclaw/openclaw"
"Tạo issue: 'Bug: browser snapshot fails on WSL2' với label bug"
"Xem kết quả CI của commit vừa push"
```

---

### 4.2 `docker-manager` — Quản lý Docker qua chat
**Developer Stack**

```bash
openclaw skills install docker-manager
```

Quản lý containers, images, compose stacks bằng câu hỏi tự nhiên thay vì gõ lệnh Docker.

Yêu cầu: Docker daemon đang chạy.

Ví dụ dùng:
```
"List tất cả containers đang chạy"
"Stop container postgres và xóa nó"
"Xem logs của container api-server 100 dòng gần nhất"
"Restart toàn bộ stack trong docker-compose.yml"
```

---

### 4.3 `cursor-cli` — Kết nối OpenClaw với Cursor AI
**Developer Stack · Trending**

```bash
openclaw skills install cursor-cli
```

Điều khiển Cursor AI editor từ chat: yêu cầu viết code, refactor, debug thông qua tmux automation. Phù hợp khi muốn agent tự mở Cursor và làm task lập trình.

Yêu cầu: Cursor đã install. tmux.

---

### 4.4 `vercel` — Deploy lên Vercel qua chat
**Developer Stack**

```bash
openclaw skills install vercel
```

Deploy, quản lý env variables, xem logs, rollback — tất cả bằng ngôn ngữ tự nhiên.

Yêu cầu: `VERCEL_TOKEN` từ vercel.com/account/tokens.

---

## NHÓM 5 — Automation & Workflow

---

### 5.1 `clawflows` — Kết nối nhiều skills thành workflow
**Workflow Orchestration**

```bash
openclaw skills install clawflows
```

Định nghĩa workflow nhiều bước, kết nối skills với nhau theo sequence hoặc condition. Ví dụ: khi nhận email từ client → tóm tắt → tạo task trong Notion → reply xác nhận.

Yêu cầu: Các skills trong flow phải được cài trước.

---

### 5.2 `n8n-workflow` — Kết nối n8n tự host
**#8 Top Tested · Automation**

```bash
openclaw skills install n8n-workflow
```

Nếu bạn đang tự host n8n, skill này cho agent **trigger, monitor, và control** workflows n8n từ chat. Kết nối 400+ integrations của n8n mà không cần code.

Yêu cầu: n8n instance đang chạy + n8n API key.

---

### 5.3 `home-assistant` — Điều khiển smart home
**Smart Home · Stable**

```bash
openclaw skills install home-assistant
```

Nói chuyện với agent để bật/tắt đèn, điều chỉnh điều hòa, check camera, set automation. Agent dùng Home Assistant REST API để gọi service.

Yêu cầu: Home Assistant instance + Long-lived access token.

```bash
echo 'HA_URL=http://homeassistant.local:8123' >> ~/.openclaw/.env
echo 'HA_TOKEN=eyJhbGc...' >> ~/.openclaw/.env
```

Ví dụ dùng:
```
"Tắt đèn phòng ngủ"
"Điều hòa phòng khách 26 độ"
"Check camera cửa trước"
"Set scene 'Good Night' lúc 23h mỗi ngày"
```

---

### 5.4 `sonoscli` — Điều khiển loa Sonos
**#10 ClawHub · 10,304 downloads**

```bash
openclaw skills install sonoscli
```

Play music, control volume, group speakers, switch rooms — tất cả từ Telegram.

Yêu cầu: Sonos speakers trên cùng network.

---

## NHÓM 6 — Communication & Media

---

### 6.1 `humanize-ai-text` — Chỉnh văn bản AI thành tự nhiên hơn
**#12 ClawHub · 8,771 downloads**

```bash
openclaw skills install humanize-ai-text
```

Khi agent viết email/báo cáo/content, skill này chỉnh lại để bỏ dấu hiệu "AI-generated": cấu trúc cứng nhắc, từ lặp lại, tone quá formal. Output tự nhiên hơn, phù hợp giọng văn của bạn.

Yêu cầu: Không có.

---

### 6.2 `elevenlabs-agent` — Voice synthesis
**Top Tested · Communication**

```bash
openclaw skills install elevenlabs-agent
```

Text-to-speech với giọng nói tự nhiên (ElevenLabs). Useful khi muốn agent đọc summary thay vì gửi text dài, hoặc gọi điện fallback khi message không được đọc.

Yêu cầu: `ELEVENLABS_API_KEY` — free tier 10,000 chars/tháng.

---

### 6.3 `meeting-prep` — Chuẩn bị họp tự động
**Productivity Stack**

```bash
openclaw skills install meeting-prep
```

Trước mỗi cuộc họp: lấy calendar details, gather relevant docs, tìm email threads liên quan, tóm tắt context. Gửi briefing 15 phút trước giờ họp.

Yêu cầu: `gog` cài trước.

---

## NHÓM 7 — Self-Hosting & Privacy

---

### 7.1 `wacli` — Swiss-army CLI utility
**#2 ClawHub · 16,415 downloads · ★37**

```bash
openclaw skills install wacli
```

Multi-purpose utility mở rộng agent với nhiều system-level capabilities: file management nâng cao, process control, system info, network utilities. Tương đương một bộ công cụ shell scripts được wrap thành skills.

Yêu cầu: Không có. Hoạt động với exec tool.

---

### 7.2 `byterover` — Multi-purpose task handler
**#3 ClawHub · 16,004 downloads · ★36**

```bash
openclaw skills install byterover
```

Task handler đa năng: kết hợp utility và development capabilities. Dùng nhiều nhất cho automation workflows phức tạp không fit vào một skill cụ thể nào.

Yêu cầu: Không có.

---

### 7.3 `auto-updater-skill` — Tự cập nhật skills
**#18 ClawHub · 6,601 downloads**

```bash
openclaw skills install auto-updater-skill
```

Tự động kiểm tra và update tất cả installed skills định kỳ. Đặt trong Heartbeat hoặc weekly Cron.

Yêu cầu: Không có.

---

## Bộ cài đặt theo profile người dùng

### Starter Pack — Bắt đầu với 5 skills cốt lõi

```bash
# Chạy lần lượt, kiểm tra từng cái
openclaw skills install capability-evolver
openclaw skills install self-improving-agent
openclaw skills install gog
openclaw skills install agent-browser
openclaw skills install summarize
```

---

### Daily Assistant Pack — Trợ lý văn phòng đầy đủ

```bash
openclaw skills install capability-evolver
openclaw skills install self-improving-agent
openclaw skills install proactive-agent
openclaw skills install gog
openclaw skills install notion
openclaw skills install summarize
openclaw skills install mission-control
openclaw skills install task-prioritizer
openclaw skills install meeting-prep
openclaw skills install weather
openclaw skills install humanize-ai-text
```

---

### Developer Pack — Cho lập trình viên

```bash
openclaw skills install capability-evolver
openclaw skills install self-improving-agent
openclaw skills install agent-browser
openclaw skills install github
openclaw skills install docker-manager
openclaw skills install cursor-cli
openclaw skills install tavily-search
openclaw skills install vercel
openclaw skills install summarize
```

---

### Smart Home Pack — Cuộc sống kết nối

```bash
openclaw skills install capability-evolver
openclaw skills install gog
openclaw skills install mission-control
openclaw skills install home-assistant
openclaw skills install sonoscli
openclaw skills install weather
openclaw skills install summarize
openclaw skills install proactive-agent
```

---

## Tổng hợp nhanh — bảng tra cứu

| Slug | Downloads | Cần API key | Dùng để |
|------|-----------|-------------|---------|
| `capability-evolver` | 35K | Không | Agent tự cải thiện theo thời gian |
| `self-improving-agent` | 16K | Không | Học từ lỗi và correction |
| `proactive-agent` | 7K | Không | Agent chủ động, không chỉ phản ứng |
| `gog` | 14K | Google OAuth | Gmail, Calendar, Drive, Docs, Sheets |
| `notion` | — | Notion API | Note, database, project board |
| `summarize` | 11K | Không | Tóm tắt text, URL, PDF |
| `mission-control` | — | Cần `gog` | Morning briefing tổng hợp |
| `task-prioritizer` | — | Không | Sắp xếp ưu tiên task |
| `obsidian` | 6K | Không | Đọc/ghi Obsidian vault |
| `agent-browser` | 12K | Không | Điều khiển browser, web automation |
| `tavily-search` | 8K | Tavily API (free) | Web search chất lượng cao |
| `weather` | 9K | OpenWeather (opt) | Thời tiết real-time |
| `github` | 11K | `gh` CLI auth | PR, Issues, Actions |
| `docker-manager` | — | Docker daemon | Quản lý containers |
| `cursor-cli` | — | Cursor install | Điều khiển Cursor AI editor |
| `vercel` | — | Vercel token | Deploy, manage projects |
| `clawflows` | — | Cần skills khác | Orchestrate multi-step workflows |
| `n8n-workflow` | — | n8n instance | Trigger n8n automations |
| `home-assistant` | — | HA token | Smart home control |
| `sonoscli` | 10K | Không | Điều khiển loa Sonos |
| `humanize-ai-text` | 9K | Không | Làm văn bản AI tự nhiên hơn |
| `elevenlabs-agent` | — | ElevenLabs API | Text-to-speech |
| `meeting-prep` | — | Cần `gog` | Chuẩn bị họp tự động |
| `wacli` | 16K | Không | System utilities đa năng |
| `byterover` | 16K | Không | Multi-purpose task handler |
| `auto-updater-skill` | 7K | Không | Tự update skills |

---

## Lưu ý an toàn trước khi cài

```bash
# Luôn xem info trước khi cài
openclaw skills info <slug>

# Cờ đỏ — cần cân nhắc kỹ:
# shell.execute + fs.read_root  → có thể đọc toàn bộ filesystem
# network.unrestricted          → gọi bất kỳ URL nào
# exec.unrestricted             → chạy lệnh shell không giới hạn

# Sau ClawHavoc cleanup, số skills còn lại đã được lọc
# nhưng vẫn nên đọc permissions trước khi cài
```

> **Nguồn**: [ClawHub Top Skills](https://clawoneclick.com/en/blog/clawhub-top-skills-2026) · [awesome-openclaw-skills](https://github.com/VoltAgent/awesome-openclaw-skills) · [BetterClaw 15 Tested](https://www.betterclaw.io/blog/best-openclaw-skills) · [open-claw.sh ranked list](https://www.open-claw.sh/blog/best-openclaw-skills-2026)
