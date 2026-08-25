# 1. Mục tiêu và sản phẩm triển khai

Tài liệu này dành cho trợ lý AI đọc trực tiếp và thực hiện việc triển khai dự án GitHub: https://github.com/oiov/vmail . Mục tiêu là triển khai vmail lên Cloudflare Workers + D1 và bật các khả năng sau:

- Thống kê tốc độ tăng trưởng của trang (today vs yesterday)
- Turnstile tùy chọn (tự động bypass khi chưa cấu hình khóa)
- Kiểm soát truy cập bằng mật khẩu trang (`PASSWORD` tùy chọn)
- Giới hạn tần suất API Key mỗi phút (có thể cấu hình bằng biến môi trường, mặc định `100`)

Hợp đồng giao diện chính sau khi triển khai:

- `GET /config` trả về: `turnstileEnabled`, `sitePasswordEnabled`, `apiRateLimitPerMinute`, `openApiEnabled`
- `GET /api/stats` trả về: `totals`, `today`, `yesterday`
- `POST /auth/unlock`, `GET /auth/status`, `POST /auth/logout`

# 2. Điều kiện tiên quyết và chuẩn bị tài khoản

Trước khi bắt đầu, hãy xác nhận:

- Đã cài đặt Node.js 18+ và pnpm
- Đã cài đặt Wrangler CLI (`npx wrangler --version` chạy được)
- Đã đăng nhập Cloudflare (`npx wrangler login`)
- Đã tạo cơ sở dữ liệu D1 trên Cloudflare và có được:
  - `D1_DATABASE_NAME`
  - `D1_DATABASE_ID`
- Đã chuẩn bị cấu hình tên miền trang `EMAIL_DOMAIN` (hỗ trợ nhiều tên miền phân cách bằng dấu phẩy)

Khuyến nghị thực hiện tất cả lệnh tại thư mục gốc của repository.

# 3. Biến môi trường bắt buộc và tùy chọn

Cần cung cấp các biến sau trong môi trường triển khai (khớp với `wrangler.toml`):

- `EMAIL_DOMAIN` (bắt buộc)
- `COOKIES_SECRET` (bắt buộc, khuyến nghị chuỗi ngẫu nhiên cường độ cao)
- `TURNSTILE_KEY` (tùy chọn)
- `TURNSTILE_SECRET` (tùy chọn)
- `PASSWORD` (tùy chọn; để trống thì trang mặc định công khai)
- `API_RATE_LIMIT_PER_MINUTE` (tùy chọn; giá trị không hợp lệ hoặc thiếu sẽ quay về `100`)
- `ENABLE_OPENAPI` (tùy chọn; mặc định tắt, chỉ khi đặt rõ là `true` mới cho phép tạo API Key và truy cập `/api/v1/*`)

Giải thích hành vi:

- Khi thiếu bất kỳ khóa nào trong `TURNSTILE_KEY`/`TURNSTILE_SECRET`, frontend lẫn backend đều vào chế độ "không cần xác minh người".
- Khi `PASSWORD` để trống, frontend không hiện cổng chặn mở khóa trang; khi có giá trị thì cần mở khóa trang trước.
- `API_RATE_LIMIT_PER_MINUTE` áp dụng cho middleware API Key v1, giới hạn theo "mỗi API Key, cửa sổ cố định mỗi phút".
- Khi `ENABLE_OPENAPI` chưa cấu hình hoặc không phải `true`, `/api/api-keys` và `/api/v1/*` sẽ đồng loạt trả về `403 OPENAPI_DISABLED`, trang `/api-docs` chỉ hiển thị thông báo.

# 4. Migration cơ sở dữ liệu và khởi tạo cơ bản

Thư mục migration D1 của dự án là: `worker/drizzle`.

Thứ tự khuyến nghị:

1. Đảm bảo cấu hình binding D1 trong `wrangler.toml` đúng.
2. Thực hiện migration tới D1 từ xa (chọn lệnh tương ứng theo môi trường Cloudflare của bạn).
3. Xác nhận các bảng mới tồn tại:
   - `daily_stats`
   - `api_rate_limits`

`api_rate_limits` dùng khóa chính phức hợp:

- `api_key_id`
- `window_start_epoch_sec`

Thiết kế này dùng để hỗ trợ đếm UPSERT nguyên tử, tránh sai lệch bộ đếm giới hạn tần suất khi có đồng thời.

# 5. Các bước build và triển khai

Thực hiện theo thứ tự sau:

1. Cài đặt phụ thuộc: `pnpm install`
2. Build cục bộ: `pnpm build`
3. Triển khai Worker (chạy wrangler deploy theo môi trường của bạn)
4. Nếu có tên miền tùy chỉnh hoặc route, hoàn tất binding route Cloudflare

Tiêu chuẩn build thành công:

- `frontend` build hoàn tất
- `worker` đóng gói không có lỗi chặn
- Cho phép tồn tại cảnh báo kích thước chunk nhưng không được có build thất bại

# 6. Danh sách nghiệm thu sau triển khai (có thể đưa trực tiếp cho AI thực hiện)

Các ca nghiệm thu tối thiểu:

1. Hợp đồng cấu hình:
   - Yêu cầu `GET /config`
   - Kiểm tra các trường trả về gồm:
     - `turnstileEnabled`
     - `sitePasswordEnabled`
     - `apiRateLimitPerMinute`
     - `openApiEnabled`
2. Hợp đồng thống kê:
   - Yêu cầu `GET /api/stats`
   - Kiểm tra cấu trúc trả về chứa `totals/today/yesterday`
3. Mật khẩu trang:
   - Khi `PASSWORD` để trống: trang truy cập trực tiếp được
   - Khi `PASSWORD` có giá trị:
     - Chưa mở khóa thì truy cập trang bị hạn chế
     - Sau khi `POST /auth/unlock` thành công thì truy cập được
     - `GET /auth/status` trả về `unlocked: true`
4. Turnstile tùy chọn:
   - Khi chưa cấu hình khóa, quy trình tạo địa chỉ email đi qua trực tiếp
   - Khi đã cấu hình khóa, cần token hợp lệ
5. Giới hạn tần suất API:
   - Dùng cùng một API Key yêu cầu liên tục API v1
   - Vượt ngưỡng trả về `429`
   - Header phản hồi chứa: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
   - Nếu `ENABLE_OPENAPI` chưa cấu hình hoặc không phải `true`: `/api/api-keys` và `/api/v1/*` trả về `403`
6. Hiển thị frontend:
   - Thẻ SiteStats trên trang chủ hiển thị tỷ lệ tăng trưởng (today vs yesterday)
   - `/api-docs` hiển thị thông báo đã vô hiệu hóa khi `openApiEnabled=false`

# 7. Các vấn đề thường gặp và chiến lược rollback

Các vấn đề thường gặp:

- Migration thất bại: ưu tiên kiểm tra binding D1, ID cơ sở dữ liệu, thư mục migration có đúng không.
- Trang liên tục bị khóa: kiểm tra `PASSWORD` đã được đặt chưa, trình duyệt có lưu cookie mở khóa không.
- Giới hạn tần suất không có tác dụng: xác nhận yêu cầu đi qua API v1 và có mang API Key.
- Trạng thái Turnstile bất thường: xác nhận frontend và backend đọc cùng một nhóm biến môi trường.

Chiến lược rollback:

1. Rollback phiên bản Worker trước (rollback mã).
2. Nếu cần rollback cấu trúc dữ liệu, thực hiện reverse migration theo chiến lược migration D1 hoặc khôi phục bản sao lưu.
3. Sau khi rollback, lặp lại các mục 1~3 trong "Danh sách nghiệm thu sau triển khai" để đảm bảo dịch vụ khả dụng.
