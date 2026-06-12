# Screenshots

Đặt 3 ảnh chụp vào thư mục này (yêu cầu của [DAY12_DELIVERY_CHECKLIST.md](../DAY12_DELIVERY_CHECKLIST.md)):

| File | Nội dung cần chụp |
|---|---|
| `render-dashboard.png` | Service trên Render ở trạng thái **Live** (tab Events/Overview) |
| `health-200.png` | Mở `https://ai-agent-production-rhdw.onrender.com/health` trên trình duyệt → thấy JSON `{"status":"ok",...}` |
| `ratelimit-429.png` | Terminal chạy vòng lặp `/ask` → thấy chuyển từ `200` sang `429` |

> Đây là phần duy nhất cần bạn tự làm (mình không truy cập được trình duyệt của bạn).
> Sau khi thêm ảnh: `git add screenshots/*.png && git commit -m "Add deployment screenshots" && git push`.
