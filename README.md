<div align="center">
  <a href="https://trendshift.io/repositories/8681" target="_blank"><img src="https://trendshift.io/api/badge/repositories/8681" alt="yesmore%2Fvmail | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>
  <h1>𝐕𝐌𝐀𝐈𝐋.𝐃𝐄𝐕</h1>
  <p><a href="/docs/github-action-tutorial_vi.md">Hướng dẫn triển khai</a>  ·  <a href="/docs/ai-deploy_vi.md">AI triển khai giúp bạn</a>  ·
  <a href="https://vmail.dev/api-docs" target="_blank">Tài liệu API</a> · <a href="https://github.com/oiov/vmail/blob/main/README_en.md">English</a> | <a href="https://github.com/oiov/vmail/blob/main/README.md">简体中文</a></p>
  <p>Dịch vụ email tạm thời triển khai bằng Cloudflare Email Worker</p>
</div>

> 🌟 Giới thiệu kênh API ổn định của **Claude Code**: [Nbility AI Gateway](https://nbility.ai/auth/register?aff=Dptp) , hỗ trợ các mô hình AI Coding phổ biến như Claude Fable 5, GPT 5.6

## 🌈 Tính năng

- 🎯 Thân thiện với quyền riêng tư, không cần đăng ký, dùng ngay
- ✈️ Hỗ trợ gửi và nhận email
- ✨ Hỗ trợ lưu mật khẩu, khôi phục hộp thư
- 😄 Hỗ trợ nhiều hậu tố tên miền
- 🔌 **API RESTful mở**, hỗ trợ gọi lập trình
- 🚀 Triển khai nhanh chóng, giải pháp thuần Cloudflare, không cần máy chủ

Nguyên lý hoạt động:

- Email worker nhận email
- Frontend (Vite + React) hiển thị email
- Lưu trữ email (Cloudflare D1)
- Gửi email bằng Resend API, MailChannels API hoặc dịch vụ gửi thư gốc của Cloudflare

## 👋 Hướng dẫn tự triển khai

Dự án này được xây dựng dựa trên Cloudflare Workers và Cloudflare D1. Bạn chỉ cần một tên miền được quản lý (host) trên Cloudflare.

### Chuẩn bị

- Tài khoản [Cloudflare](https://dash.cloudflare.com/) và tên miền được quản lý trên Cloudflare
- Cài đặt môi trường [Node.js](https://nodejs.org) (phiên bản >= 22.x) và [pnpm](https://pnpm.io/installation) trên máy

### Triển khai tự động (khuyến nghị)

Dự án đã bao gồm một workflow GitHub Action được cấu hình sẵn, giúp bạn tự động triển khai ứng dụng Vmail lên Cloudflare.

Vui lòng tham khảo [Hướng dẫn triển khai tự động bằng GitHub Action](/docs/github-action-tutorial.md) để biết chi tiết.

### Các bước triển khai thủ công

1.  **Clone dự án về máy**
    ```bash
    git clone https://github.com/oiov/vmail
    cd vmail
    pnpm install
    ```

2.  **Tạo cơ sở dữ liệu Cloudflare D1**
    Tạo cơ sở dữ liệu D1 trong bảng điều khiển Cloudflare hoặc bằng Wrangler CLI.

3.  **Cấu hình `wrangler.toml`**
    Thay các placeholder `${...}` trong file `wrangler.toml` ở thư mục gốc bằng thông tin Cloudflare và D1 của bạn. Bạn cũng có thể thiết lập các giá trị này thông qua biến môi trường của Cloudflare Pages.

4.  **Build và triển khai**
    ```bash
    # Build ứng dụng frontend
    pnpm run build

    # Triển khai lên Cloudflare
    pnpm run deploy
    ```
    Wrangler sẽ tự động xử lý việc triển khai tài nguyên tĩnh của frontend và Worker, đồng thời áp dụng các migration cơ sở dữ liệu theo cấu hình.

5.  **Cấu hình định tuyến email**
    Trong giao diện quản lý tên miền Cloudflare, vào `Email` -> `Email Routing` -> `Routes`, thiết lập quy tắc `Catch-all` để chuyển tất cả email gửi tới tên miền của bạn với thao tác `Send to a Worker`, rồi chọn Worker vừa triển khai.

### Biến môi trường

Khi triển khai lên Cloudflare Workers bằng GitHub Actions, bạn cần cấu hình các biến môi trường sau:

-   `D1_DATABASE_NAME`: Tên cơ sở dữ liệu D1 của bạn.
-   `D1_DATABASE_ID`: ID cơ sở dữ liệu D1 của bạn.
-   `COOKIES_SECRET`: Khóa dùng để ký Cookie.
-   `EMAIL_DOMAIN`: Tên miền email của bạn, ví dụ `example.com,example.net`.
-   `TURNSTILE_KEY`: Khóa site Turnstile của bạn, tùy chọn.
-   `TURNSTILE_SECRET`: Khóa bí mật Turnstile của bạn, tùy chọn.
-   `PASSWORD`: Mật khẩu truy cập trang web (tùy chọn).
-   `API_RATE_LIMIT_PER_MINUTE`: Giới hạn yêu cầu API mỗi phút (tùy chọn, mặc định 100).
-   `SEND_CHANNEL`: Kênh gửi thư, một trong `resend`, `mailchannels`, `cloudflare`; nếu không cấu hình thì ẩn tính năng gửi thư. Giá trị cũ `send_email` vẫn tương thích nhưng đã không còn được khuyến khích.
-   `SENDER_EMAIL`: Địa chỉ người gửi cố định, phải là địa chỉ được nhà cung cấp cho phép hoặc xác thực; hộp thư tạm thời chỉ dùng làm `Reply-To`.
-   `MAILBOX_TOKEN_SECRET`: Khóa ký token ủy quyền gửi thư của hộp thư, bắt buộc khi bật gửi thư và nên cấu hình bằng Wrangler secret.
-   `RESEND_API_KEY`: Khóa API Resend, chỉ cần khi `SEND_CHANNEL=resend`, cấu hình bằng Wrangler secret.
-   `MAILCHANNELS_API_KEY`: Khóa API MailChannels, chỉ cần khi `SEND_CHANNEL=mailchannels`, cấu hình bằng Wrangler secret.
-   `SEND_RATE_LIMIT_PER_MINUTE`: Số email tối đa mỗi hộp thư gửi trong một phút (tùy chọn, mặc định 3).
-   `SEND_IP_RATE_LIMIT_PER_MINUTE`: Số email tối đa mỗi IP gửi trong một phút (tùy chọn, mặc định 10).
-   `SHOW_AFF`: Có hiển thị popup và liên kết quảng cáo hay không (tùy chọn, đặt `true` để bật).
-   `ENABLE_OPENAPI`: Có bật chức năng OpenAPI hay không (tùy chọn, mặc định `false`; chỉ khi đặt rõ là `true` mới cho phép tạo API Key và truy cập `/api/v1/*`).

Không ghi khóa gửi thư vào `wrangler.toml`, hãy dùng:

```bash
pnpm exec wrangler secret put MAILBOX_TOKEN_SECRET
pnpm exec wrangler secret put RESEND_API_KEY       # khi dùng Resend
pnpm exec wrangler secret put MAILCHANNELS_API_KEY # khi dùng MailChannels
```

Khi dùng dịch vụ gửi thư gốc của Cloudflare Worker, đặt `SEND_CHANNEL` là `cloudflare` và cấu hình `SENDER_EMAIL` cùng `MAILBOX_TOKEN_SECRET`; không cần `RESEND_API_KEY` hoặc `MAILCHANNELS_API_KEY`. Ngoài ra, cần bật Cloudflare Email Routing cho tên miền đó. Binding `[[send_email]]` có tên `SEND_EMAIL` trong `wrangler.toml` là cấu hình cố định của Cloudflare, không cần đổi tên.

## 🔨 Chạy và gỡ lỗi cục bộ

1.  **Sao chép file biến môi trường**
    ```bash
    # Lệnh này tạo file biến môi trường cục bộ, wrangler dev sẽ tự động tải
    cp .env.example .env
    ```

2.  **Điền biến môi trường cục bộ**
    Điền các biến môi trường cần thiết vào file `.env`, đặc biệt là `D1_DATABASE_ID`... Bạn cần tạo cơ sở dữ liệu D1 trên Cloudflare để phát triển cục bộ.

3.  **Khởi động máy chủ phát triển**
    ```bash
    pnpm run dev
    ```
    Lệnh này sẽ đồng thời khởi động máy chủ phát triển Vite (frontend) và môi trường Wrangler Worker cục bộ.

## 📖 Tài liệu API

Vmail cung cấp API RESTful đầy đủ, hỗ trợ tạo hộp thư tạm thời và truy vấn hộp thư đến bằng chương trình.

### Lấy API Key

Truy cập [trang Tài liệu API](https://vmail.dev/api-docs) để tạo API Key miễn phí.

### API Endpoints

| Phương thức | Endpoint                                    | Mô tả                             |
| ----------- | ------------------------------------------- | --------------------------------- |
| `POST`      | `/api/v1/mailboxes`                         | Tạo hộp thư tạm thời              |
| `GET`       | `/api/v1/mailboxes/:id`                     | Lấy thông tin hộp thư             |
| `GET`       | `/api/v1/mailboxes/:id/messages`            | Lấy hộp thư đến (hỗ trợ phân trang) |
| `GET`       | `/api/v1/mailboxes/:id/messages/:messageId` | Lấy chi tiết email                |
| `DELETE`    | `/api/v1/mailboxes/:id/messages/:messageId` | Xóa email                         |

### Bắt đầu nhanh

```bash
# 1. Tạo hộp thư tạm thời
curl -X POST https://vmail.dev/api/v1/mailboxes \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json"

# Phản hồi: { "data": { "id": "abc123", "address": "random@domain.com", ... } }

# 2. Truy vấn hộp thư đến
curl https://vmail.dev/api/v1/mailboxes/abc123/messages \
  -H "X-API-Key: your-api-key"

# 3. Lấy chi tiết email
curl https://vmail.dev/api/v1/mailboxes/abc123/messages/msg_001 \
  -H "X-API-Key: your-api-key"
```

Tài liệu đầy đủ: [https://vmail.dev/api-docs](https://vmail.dev/api-docs)

## 📝 Giấy phép

GNU General Public License v3.0

## Lịch sử Star

[![Star History Chart](https://api.star-history.com/svg?repos=oiov/vmail&type=Date)](https://star-history.com/#oiov/vmail&Date)
