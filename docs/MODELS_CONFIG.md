# Hướng Dẫn Cấu Hình Model (Phiên bản mới)

Trong các phiên bản mới của OpenClaw, cấu trúc file `openclaw.json` đã thay đổi để tách biệt rõ ràng giữa **Khả năng của Model (Hardware Limits)** và **Cách Agent sử dụng Model (Agent Binding)**.

Chính vì vậy, nếu bạn cố gắng khai báo `contextWindow` hoặc `maxTokens` vào bên trong mục `agents.defaults.models` thì hệ thống sẽ báo lỗi Schema và tự động xoá đi.

Dưới đây là cách cấu hình đúng chuẩn:

## 1. Cấu hình Khả năng của Model (Tại `models.providers`)

Tất cả các thông số vật lý của một Model (như context window, max tokens, chi phí, v.v.) phải được khai báo ở block `"models"` nằm ngoài cùng của file (root level). 

Ví dụ:
```json
  "models": {
    "mode": "merge",
    "providers": {
      "custom-localhost-20128": {
        "baseUrl": "http://localhost:20128/v1",
        "api": "openai-completions",
        "apiKey": "sk_9router",
        "models": [
          {
            "id": "nvidia/moonshotai/kimi-k2.5",
            "name": "Kimi k2.5",
            "contextWindow": 128000,     // <--- ĐẶT CONTEXT WINDOW Ở ĐÂY
            "maxTokens": 4096,           // <--- ĐẶT MAX TOKENS Ở ĐÂY
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "reasoning": false
          }
        ]
      }
    }
  }
```

## 2. Gắn Model cho Agent (Tại `agents.defaults`)

Sau khi đã định nghĩa toàn bộ sức mạnh của Model ở phần `models.providers` bên trên, bạn mới vào phần `agents` để quyết định xem Agent nào sẽ dùng Model nào.

Ở phần này, bạn **CHỈ** được phép đặt các bí danh (`alias`) hoặc khai báo model `primary` mà thôi, TUYỆT ĐỐI không đặt `contextWindow` ở đây:

```json
  "agents": {
    "defaults": {
      "workspace": "/Users/tungnt/.openclaw/workspace",
      "model": {
        "primary": "custom-localhost-20128/gc/gemini-3-flash-preview"
      },
      "models": {
        "custom-localhost-20128/nvidia/moonshotai/kimi-k2.5": {
          "alias": "nvidia_kimi-k2.5"   // <--- CHỈ ĐẶT ALIAS Ở ĐÂY
        }
      }
    }
  }
```

## Tổng kết
- **Muốn chỉnh maxTokens / contextWindow:** Kéo xuống dưới cùng tìm mục `"models": { "providers": ... }`.
- **Muốn chọn model nào chạy chính:** Kéo lên mục `"agents": { "defaults": { "model": { "primary": ... } } }`.
