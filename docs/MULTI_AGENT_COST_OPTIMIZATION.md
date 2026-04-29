# Tối ưu hóa chi phí với Multi-Agent & Model Routing trong OpenClaw

Tài liệu này hướng dẫn cách cấu hình OpenClaw để tự động chọn model phù hợp cho từng loại tác vụ (Semantic Routing ở tầng ứng dụng). 

Nguyên tắc cốt lõi: **Việc khó giao model cao cấp (tốn phí), việc dễ giao model nhẹ (giá rẻ hoặc chạy local miễn phí).**

---

## 1. Cơ chế Multi-Agent Model Routing

Thay vì dùng một proxy tầng mạng (như 9Router) để tự động "đoán" độ khó của câu hỏi (điều này thường gây độ trễ cao và thiếu chính xác), OpenClaw phân bổ model dựa trên **ngữ cảnh và tính chất của tác vụ**.

Hệ thống cho phép gán (override) các model chuyên biệt cho:
1. **Lệnh Chat trực tiếp (Main Session):** Giao tiếp hằng ngày với User.
2. **Tác vụ nền (Cron Jobs):** Chạy ngầm theo lịch định kỳ.
3. **Chức năng tự thức dậy (Heartbeat):** Thường xuyên ping để kiểm tra hệ thống hoặc thông báo mới.
4. **Luồng công việc (Task Flows):** Chuỗi công việc kết hợp nhiều Agent.

---

## 2. Cấu hình Model nhẹ cho Heartbeat (Tiết kiệm Token)

**Heartbeat** là cơ chế Agent tự động "thức dậy" định kỳ (ví dụ mỗi 30 phút) để kiểm tra các thông báo, check email, check lịch. 
Bởi vì Heartbeat chạy liên tục 24/7, nếu dùng model Pro sẽ đốt token cực kỳ khủng khiếp. Do đó, ta **bắt buộc** phải cấu hình Heartbeat dùng model siêu nhẹ (ví dụ `gemini-3-flash` hoặc các model giá rẻ).

**Các lệnh cấu hình qua CLI:**

```bash
# 1. Cấu hình chu kỳ thức dậy (ví dụ 30 phút một lần)
openclaw config set agents.defaults.heartbeat.enabled true
openclaw config set agents.defaults.heartbeat.every "30m"

# 2. Quan trọng: Gán model Local (hoàn toàn miễn phí) cho riêng Heartbeat
openclaw config set agents.defaults.heartbeat.model "ollama/gemma3:4b"

# 3. Tối ưu Token Context
# Bật isolatedSession để không load lại toàn bộ lịch sử chat cũ mỗi khi thức dậy
openclaw config set agents.defaults.heartbeat.isolatedSession true
openclaw config set agents.defaults.heartbeat.lightContext true

# Khởi động lại Gateway để áp dụng cấu hình
openclaw gateway restart
```

---

## 3. Gán Model linh hoạt cho Cron Jobs

Đối với các tác vụ đặt lịch (Cron), anh có thể phân bổ model tùy theo "độ khó" của task đó ngay trong câu lệnh.

### A. Tác vụ đơn giản (Dùng Local Model hoặc Free Model)
Ví dụ: Kiểm tra trạng thái máy chủ, ping API, lấy giá thời tiết... Những việc này không cần tư duy cao cấp.
```bash
openclaw cron add \
  --name "server-health" \
  --every "2h" \
  --session isolated \
  --model "ollama/llama3.2:latest" \
  --message "Fetch https://myapp.com/health. Nếu API lỗi, báo ngay qua Telegram."
```

### B. Tác vụ phức tạp (Dùng Model Pro)
Ví dụ: Tổng hợp báo cáo, phân tích số liệu tài chính, review code. Những tác vụ này cần khả năng suy luận mạnh mẽ.
```bash
openclaw cron add \
  --name "morning-briefing" \
  --cron "0 7 * * 1-5" \
  --session isolated \
  --model "gc/gemini-3-pro-preview" \
  --message "Tổng hợp các task chưa hoàn thành hôm qua và lên kế hoạch công việc hôm nay."
```

---

## 4. Phân rã công việc với Multi-Agent Task Flows

Trong OpenClaw, khi phải đối mặt với một chuỗi công việc dài, anh có thể cấu hình một **Task Flow** (Luồng công việc) có sự tham gia của nhiều Model/Agent khác nhau:

1. **Agent Thu thập (Model Rẻ/Local):** Cào dữ liệu từ Website, đọc lướt hàng nghìn trang tài liệu để trích xuất text thô.
2. **Agent Phân tích (Model Xịn/Pro):** Nhận đoạn text đã được tinh lọc ở bước 1, phân tích logic, viết báo cáo hoặc sửa lỗi code.
3. **Agent Format (Model Rẻ):** Format lại báo cáo thành Markdown hoặc bắn qua Telegram.

Cơ chế này đảm bảo Model Xịn chỉ phải đọc nội dung đã được xử lý tinh gọn, tiết kiệm tối đa lượng Token input khổng lồ so với việc bắt Model Xịn tự đi cào dữ liệu thô.

*(Chi tiết tạo luồng có thể tham khảo thêm tại lệnh `openclaw task --help`)*.

---

### Tổng kết

Thay vì trông chờ vào một phép màu "Semantic Router" từ Proxy để đoán câu hỏi, việc **Setup thẳng Model vào từng loại công việc** như OpenClaw giúp anh:
- **Kiểm soát 100% chi phí:** Không bao giờ sợ model xịn bị lạm dụng để check mail hay check health server.
- **Bảo đảm chất lượng:** Luôn chắc chắn các task phân tích code, viết lách sẽ được giao đúng cho thợ chính.
- **Bảo mật ngữ cảnh (Context):** Tách bạch bộ nhớ giữa các Agent, giúp model chính không bị "ngáo" do nhồi nhét quá nhiều thông tin rác từ các cron jobs thu thập.
