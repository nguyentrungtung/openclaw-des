# OpenClaw AI Agent — Tài Liệu Kỹ Thuật Toàn Diện

> **Version**: OpenClaw 2026.4.21 (stable, tháng 4/2026)
> **Đối tượng**: Kỹ sư/developer có kinh nghiệm với Node.js, automation systems

---

## Mục lục

| File | Nội dung |
|------|----------|
| [01-architecture.md](01-architecture.md) | Kiến trúc tổng quan, định nghĩa thuật ngữ, workspace files |
| [02-installation.md](02-installation.md) | Prerequisites, cài đặt, onboard wizard, cấu trúc thư mục, lỗi thường gặp |
| [03-config.md](03-config.md) | openclaw.json đầy đủ, CLI config, model providers, routing strategy |
| [04-automation.md](04-automation.md) | Cron jobs, Heartbeat, Hooks — decision matrix và syntax đầy đủ |
| [05-browser.md](05-browser.md) | Browser modes, CLI commands, Playwright, snapshot refs, workflow thực tế |
| [06-skills-plugins.md](06-skills-plugins.md) | Skills vs Plugins, ClawHub, quản lý, security checklist |
| [07-channels.md](07-channels.md) | Telegram setup, Discord, WhatsApp và các channel khác |
| [08-security.md](08-security.md) | Security hardening, audit, backup, update |
| [09b-9router.md](09b-9router.md) | **9Router** — fallback tự động: Paid API → NVIDIA Free → Ollama Local |
| [10-workspace-templates.md](10-workspace-templates.md) | SOUL.md, AGENTS.md, USER.md, MEMORY.md, HEARTBEAT.md templates thực tế |
| [11-suggested-skills.md](11-suggested-skills.md) | **26 skills cộng đồng hay dùng nhất** — phân loại theo use case, CLI install |
| [QUICKSTART.md](QUICKSTART.md) | Từ zero đến chạy được trong 15 phút |

---

## Tổng quan hệ sinh thái

```
OpenClaw Gateway (self-hosted Node.js)
    ├── Channels: Telegram · Discord · WhatsApp · Signal · iMessage
    ├── Agents: SOUL.md + AGENTS.md + MEMORY.md + workspace files
    ├── Tools: browser · exec · web_search · read/write · MCP
    ├── Automation: Cron · Heartbeat · Hooks · Task Flow
    └── LLM Layer
          └── 9Router (port 20128) ← proxy trung gian
                ├── Tier 1: Gemini Pro / Claude (paid API key)
                ├── Tier 2: NVIDIA NIM free (Nemotron, Kimi K2.5...)
                └── Tier 3: Ollama local (Llama, Qwen — luôn sẵn sàng)
```

## Chiến lược dùng model với 9Router

| Tier | Provider | Chi phí | Khi nào dùng |
|------|----------|---------|-------------|
| **1** | Gemini Pro / Claude API | Trả phí theo token | Mặc định, chất lượng cao |
| **2** | NVIDIA NIM free | $0 (1000 credits free) | Khi tier 1 hết quota |
| **3** | Ollama local | $0, không quota | Luôn có, không bao giờ fail |

9Router tự động chuyển tier khi nhận lỗi 429/503 — OpenClaw không cần biết.

---

## Nguồn tài liệu chính thức

- [OpenClaw Docs](https://docs.openclaw.ai)
- [GitHub: openclaw/openclaw](https://github.com/openclaw/openclaw)
- [9Router: decolua/9router](https://github.com/decolua/9router)
- [NVIDIA NIM free API](https://build.nvidia.com)
- [ClawHub Skills Registry](https://clawhub.ai)
- [OpenRouter Integration](https://openrouter.ai/docs/guides/coding-agents/openclaw-integration)
