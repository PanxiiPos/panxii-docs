# SLO VÀ OBSERVABILITY SPEC

## 1. SLO đề xuất để chốt theo capacity

Các giá trị dưới đây là baseline cần được đo trên VPS staging rồi chốt chính thức:

| SLI | Baseline đề xuất |
|---|---|
| API availability | >= 99.5%/tháng |
| Read API p95 | <= 500 ms |
| Invoice sync p95 | <= 1.5 s khi dependency healthy |
| Pull sync p95 | <= 1.5 s |
| Message publish confirm p95 | <= 1 s |
| Outbox pending age | <= 5 phút bình thường |
| Backup success | 100% theo lịch |
| DLQ | 0 message không được xử lý |

## 2. Metrics

- Request count/status/latency theo endpoint và tenant-safe dimensions.
- Sync applied/rejected/duplicate/conflict.
- Outbox pending age/count, publish confirm latency.
- RabbitMQ queue depth, unacked, retry, DLQ.
- MySQL connection pool, slow query, lock/deadlock, disk.
- SignalR connected/reconnect/pull lag.
- SQLite recovery/crash/outbox pending trên mobile, không gửi PII.
- Printer success/failure/reprint latency.

## 3. Alert baseline

- 5xx > 2% trong 5 phút.
- Sync failure > 1% trong 10 phút.
- Outbox pending age > 10 phút.
- DLQ > 0 message.
- Disk < 20% hoặc certificate còn < 14 ngày.
- Backup fail một lần là warning, fail hai lịch liên tiếp là critical.
- Cross-tenant test fail là release blocker.

## 4. Correlation

Mọi API request, outbox event, RabbitMQ message, audit record và mobile error telemetry dùng `CorrelationId`. Redact token, PIN, password, SQLCipher key và PII.

## 5. Error budget

Khi SLO vượt error budget, dừng feature rollout và ưu tiên reliability work. Không dùng retry để che latency hoặc failure metric.

## 6. Availability monitor bên ngoài VPS

**Nói ngắn gọn:** backup/host alert đã tự chạy. Chỉ cần thêm một dịch vụ
bên ngoài ping URL public và báo khi mất kết nối; không phải cài thêm
monitoring stack trên VPS. Owner tạo tài khoản dịch vụ và chọn nơi nhận báo,
không cần tự quản lý dashboard metrics phức tạp.

Host monitor/Discord trên VPS theo dõi dependency, disk, backup và service.
Nếu cả VPS mất điện/mạng thì chính monitor đó không thể gửi cảnh báo; cần
thêm dịch vụ độc lập, ví dụ UptimeRobot hoặc Better Stack, do owner quản lý.

1. Tạo HTTPS monitor cho `https://panxii.duckdns.org/api/v1/health`, GET,
   yêu cầu HTTP 200 và TLS hợp lệ. Không dùng `/health/ready` private,
   Swagger, token hoặc tài khoản ứng dụng.
2. Chọn chu kỳ theo gói dịch vụ; đề xuất 1 phút nếu hỗ trợ, nếu gói miễn phí
   chỉ có 5 phút thì ghi rõ độ trễ. Báo khi lỗi hai lượt liên tiếp và khi phục
   hồi; bật cảnh báo hạn certificate nếu nhà cung cấp hỗ trợ.
3. Owner cấu hình kênh email/Discord trong giao diện dịch vụ; không đưa
   webhook/token vào chat/repo. Thử notification của nhà cung cấp trước,
   rồi kiểm tra một monitor thử có endpoint lỗi; không dừng production để thử.
4. Ghi thời gian nhận alert và recovery alert, người nhận, interval, timeout
   và ngưỡng vào evidence release. Chưa có cấu hình/alert evidence thì gate
   external monitoring vẫn pending.

Public health chỉ là probe khả dụng HTTP/API, không thay kiểm tra
MySQL/RabbitMQ, backup hoặc correctness trong host monitor.
