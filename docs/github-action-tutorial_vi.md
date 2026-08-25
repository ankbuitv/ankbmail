# Hướng dẫn triển khai tự động bằng GitHub Action

Dự án này đã bao gồm file workflow GitHub Action được cấu hình sẵn (`.github/workflows/deploy.yml`), giúp bạn tự động triển khai ứng dụng Vmail lên Cloudflare Workers.

## Chuẩn bị

Trước khi bắt đầu, hãy đảm bảo bạn đã hoàn thành các bước trong [hướng dẫn triển khai nhận email](/docs/receive-tutorial.md), đặc biệt là cấu hình Cloudflare và Turso.

## Cấu hình GitHub Secrets

Để GitHub Actions có thể truy cập an toàn vào tài khoản Cloudflare và cơ sở dữ liệu của bạn, bạn cần cấu hình các thông tin nhạy cảm sau làm Secrets của repository GitHub.

Truy cập trang repository GitHub của bạn, nhấp `Settings` -> `Secrets and variables` -> `Actions`, sau đó thêm các `Repository secrets` sau:

| Secret Name              | Mô tả                                                                                                                                                            | Giá trị ví dụ                          |
| :----------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------- |
| `CF_API_TOKEN`           | Cloudflare API Token, dùng để ủy quyền thao tác Wrangler. Hãy đảm bảo Token có quyền chỉnh sửa Workers và cơ sở dữ liệu D1.                                       | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`     |
| `CF_ACCOUNT_ID`          | ID tài khoản Cloudflare của bạn, có thể tìm thấy ở bên phải trang chủ Cloudflare Console.                                                                        | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`     |
| `D1_DATABASE_ID`         | ID cơ sở dữ liệu D1 của bạn.                                                                                                                                     | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `D1_DATABASE_NAME`       | Tên cơ sở dữ liệu D1 của bạn.                                                                                                                                    | `vmail`                                |
| `EMAIL_DOMAIN`           | Tên miền email của bạn, nếu có nhiều tên miền thì phân cách bằng dấu phẩy.                                                                                       | `vmail.dev,example.com`                |
| `COOKIES_SECRET`         | Khóa dùng để mã hóa Cookie, hãy đặt một chuỗi ngẫu nhiên đủ phức tạp.                                                                                             | `a-very-strong-and-random-secret`      |
| `TURNSTILE_KEY`          | Tùy chọn, Cloudflare Turnstile Site Key.                                                                                                                         | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`       |
| `TURNSTILE_SECRET`       | Tùy chọn, Cloudflare Turnstile Secret Key.                                                                                                                       | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`       |
| `PASSWORD`               | Tùy chọn, mật khẩu dùng để truy cập trang web Vmail.                                                                                                             | `password`                             |
| `API_RATE_LIMIT_PER_MINUTE` | Tùy chọn, giới hạn yêu cầu API mỗi phút, mặc định là `100`.                                                                                                    | `100`                                  |
| `ENABLE_OPENAPI`         | Tùy chọn, có bật OpenAPI hay không; mặc định tắt, chỉ khi đặt rõ là `true` mới cho phép tạo API Key và truy cập `/api/v1/*`.                                     | `true`                                 |
| `SEND_CHANNEL`           | Tùy chọn, kênh gửi thư: `resend`, `mailchannels` hoặc `cloudflare`; để trống nghĩa là không bật gửi thư. Giá trị cũ `send_email` chỉ dùng cho tương thích.         | `cloudflare`                            |
| `SENDER_EMAIL`           | Bắt buộc khi bật gửi thư, địa chỉ người gửi cố định được nhà cung cấp cho phép hoặc xác thực.                                                                    | `noreply@example.com`                  |
| `MAILBOX_TOKEN_SECRET`   | Bắt buộc khi bật gửi thư, khóa ngẫu nhiên dùng để ký ủy quyền gửi thư của hộp thư.                                                                               | `another-strong-random-secret`         |
| `RESEND_API_KEY`         | Bắt buộc khi `SEND_CHANNEL=resend`, workflow sẽ ghi nó vào Cloudflare Worker secret.                                                                             | `re_xxxxxxxxx`                         |
| `MAILCHANNELS_API_KEY`   | Bắt buộc khi `SEND_CHANNEL=mailchannels`, workflow sẽ ghi nó vào Cloudflare Worker secret.                                                                       | `xxxxxxxx`                             |
| `SEND_RATE_LIMIT_PER_MINUTE` | Tùy chọn, số email tối đa mỗi hộp thư gửi trong một phút, mặc định `3`.                                                                                       | `3`                                    |
| `SEND_IP_RATE_LIMIT_PER_MINUTE` | Tùy chọn, số email tối đa mỗi IP gửi trong một phút, mặc định `10`.                                                                                    | `10`                                   |

Khi dùng dịch vụ gửi thư gốc của Cloudflare Worker, thêm và đặt các Secrets sau:

- `SEND_CHANNEL=cloudflare`
- `SENDER_EMAIL=<email của tên miền được phép gửi thư>`
- `MAILBOX_TOKEN_SECRET=<chuỗi ngẫu nhiên đủ dài>`

Kênh này không cần `RESEND_API_KEY` hoặc `MAILCHANNELS_API_KEY`. Trước khi triển khai cũng cần bật Email Routing cho tên miền email trong Cloudflare; `[[send_email]]` / `SEND_EMAIL` trong repository là tên binding gốc của Cloudflare, không cần sửa đổi.

## Kích hoạt triển khai tự động

Sau khi cấu hình xong các Secrets trên, mỗi khi bạn push mã lên nhánh `main`, GitHub Action sẽ tự động kích hoạt và thực hiện các bước sau:

1.  **Kiểm tra thông tin xác thực**: Xác nhận `CF_API_TOKEN` và `CF_ACCOUNT_ID` đã được thiết lập.
2.  **Cài đặt và build**: Cài đặt phụ thuộc pnpm và build ứng dụng frontend cùng Worker.
3.  **Migration cơ sở dữ liệu**: Tự động áp dụng các script migration trong thư mục `worker/drizzle` vào cơ sở dữ liệu D1 của bạn.
4.  **Triển khai ứng dụng**: Triển khai ứng dụng đã build lên Cloudflare.

Bạn cũng có thể kích hoạt triển khai thủ công trong tab `Actions` của repository GitHub.

## Lưu ý

- File workflow (`.github/workflows/deploy.yml`) sẽ tự động đọc cấu hình từ Secrets của bạn và áp dụng vào file `wrangler.toml`, **không** ghi trực tiếp thông tin nhạy cảm vào file `wrangler.toml`.
- Nếu cấu trúc cơ sở dữ liệu của bạn thay đổi (ví dụ, sửa đổi `worker/src/database/schema.ts`), hãy nhớ tạo file migration mới và commit vào kho mã, GitHub Action sẽ tự động áp dụng cập nhật cho bạn.
