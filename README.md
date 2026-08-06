# Vibao — bản nâng cấp (server thật + tool search + limit)

## Đã thay đổi gì so với bản cũ?

1. **Không còn key AI trong app nữa.** Trước đây key Gemini/Mistral/Groq bị bake thẳng vào APK — ai cài app cũng xài chung key của bạn. Giờ app chỉ gọi tới server riêng (Cloudflare Worker), server mới giữ key thật.
2. **Đăng nhập bằng Gmail.** Mỗi tài khoản có hạn mức token/ngày riêng, lưu trên server (Cloudflare KV), không thể lách bằng cách xoá app/cài lại.
3. **Fallback nhiều key tự động.** Server thử lần lượt: 4 key Cerebras → 4 key Gemini → Groq → Mistral. Key nào lỗi/hết quota, tự chuyển key kế mà người dùng không thấy gián đoạn.
4. **Tool search thật cho AI (function calling).** Thay vì regex bắt từ "tóm tắt" rồi tự search, giờ AI tự quyết định lúc nào cần tra web (tin tức, giá cả, sự kiện mới...) rồi tự tóm tắt câu trả lời — đúng kiểu agentic.
5. **System prompt** cho AI biết mình là "Vibao", tính cách, cách dùng tool.
6. **UI đẹp hơn:** màn hình đăng nhập, thanh hiển thị hạn mức còn lại, tag "🔎 đã tìm web" trên tin nhắn có dùng tool, màu sắc/gradient mới.
7. **Lệnh mới:** `/quota` (xem hạn mức), `/clear` (xoá lịch sử đoạn chat), `/help` (danh sách lệnh).

## Cấu trúc

```
vibao-capacitor-v2/     ← app (Capacitor, giữ nguyên cách build cũ)
vibao-worker/           ← backend mới (Cloudflare Worker) — bạn cần deploy cái này
```

## Các bước cần làm (theo thứ tự)

### 1. Deploy backend trước
Xem chi tiết trong `vibao-worker/README.md`. Tóm tắt:
```bash
cd vibao-worker
npm install
npx wrangler login
npx wrangler kv namespace create VIBAO_KV   # rồi dán id vào wrangler.toml
npx wrangler secret put CEREBRAS_KEY_1      # ... và các key còn lại
npx wrangler deploy
```
Sau khi deploy xong bạn sẽ có URL dạng `https://vibao-api.xxx.workers.dev`.

### 2. Tạo Google Client ID
Xem mục 5 trong `vibao-worker/README.md`. Ra được 1 chuỗi dạng `xxxx.apps.googleusercontent.com`.

### 3. Cấu hình 2 giá trị đó ở GitHub Secrets của repo app
Vào repo GitHub → Settings → Secrets and variables → Actions → New repository secret:
- `VIBAO_API_BASE_URL` = URL worker ở bước 1
- `GOOGLE_CLIENT_ID` = Client ID ở bước 2

Workflow build (`.github/workflows/build.yml`) sẽ tự thay 2 placeholder này vào `www/index.html` lúc build APK, giống cách bake key kiểu cũ nhưng giờ không còn AI key nào lộ ra ngoài nữa.

### 4. Test trước khi build APK (khuyên dùng)
Bạn có thể mở thẳng `www/index.html` bằng một static server (vd `npx serve www`) và tự tay sửa 2 dòng:
```js
const API_BASE_URL = 'https://vibao-api.xxx.workers.dev';
const GOOGLE_CLIENT_ID = 'xxxx.apps.googleusercontent.com';
```
để test đăng nhập + chat hoạt động trước khi build APK thật.

### 5. Build APK như cũ
Push code lên `main`, GitHub Actions tự chạy, tải APK ở tab Actions → Artifacts.

## Lưu ý về Google Sign-In trong WebView (APK)

Google Identity Services (JS SDK dùng trong `index.html`) chạy tốt trên trình duyệt / PWA, nhưng **có thể bị Google chặn trong WebView đóng gói** (chính sách "Disallowed User Agent"). Nếu gặp lỗi này khi test trên APK thật, cách xử lý chuẩn là dùng plugin native `@capacitor-community/google-auth` thay vì JS SDK. Cứ build thử bản hiện tại trước — nếu đăng nhập lỗi trên APK (nhưng chạy được trên trình duyệt), báo tui để tui chuyển qua plugin native, không phải sửa gì nhiều ở phần backend.

## Giới hạn hiện tại (để bạn biết trước)

- Quota tính theo ước lượng token nếu provider không trả `usage` (Gemini luôn trả, một số OpenAI-style cũng trả — đủ chính xác để giới hạn).
- Tool search dùng DuckDuckGo (free, không key) — đủ dùng cho cá nhân, không nên kỳ vọng chất lượng như Google Search thật.
- Lịch sử đoạn chat vẫn lưu ở máy (localStorage theo thiết bị), chưa đồng bộ giữa các máy dù cùng 1 Gmail — nếu cần đồng bộ nhiều thiết bị thì sau này chuyển sang lưu ở KV/D1 theo user.
