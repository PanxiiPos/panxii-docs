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
