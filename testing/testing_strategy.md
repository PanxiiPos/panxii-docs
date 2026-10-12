# TESTING STRATEGY - POS OFFLINE-FIRST

## 1. Mục tiêu chất lượng

Kiểm thử tập trung vào các rủi ro lớn nhất: cách ly tenant, invoice bất biến, reverse entry, retry offline không duplicate, kill-switch, feature flag server-side và đồng bộ nhiều thiết bị.

## 2. Test pyramid

| Tầng | Nội dung | Tỷ trọng định hướng |
|---|---|---:|
| Unit | Domain rules, calculator, authorization, error code | Cao |
| Integration | EF Core/MySQL, RabbitMQ, transaction, outbox | Trung bình |
| API contract | OpenAPI schema, status/error response | Trung bình |
| E2E | Flutter + API + broker + DB | Thấp nhưng critical |
| Load/chaos | Sync, reconnect, broker delay, DB contention | Theo release |

## 3. Unit test bắt buộc

### Invoice calculator

- `Subtotal` bằng tổng `Quantity * UnitPrice`.
- Discount và rounding đúng thứ tự; giá sản phẩm/unit price phải > 0.
- Tax calculation chưa thuộc v1; request phải giữ `TaxRate`/`TaxAmount` bằng 0.
- Decimal không dùng floating point.
- Reverse invoice đảo dấu quantity, line total và total; tax vẫn bằng 0 trong v1.
- Header lệch detail trả `ERR_INVOICE_TOTAL_INVALID`.
- Quantity bằng zero, âm ở Sale, quá precision bị từ chối.

### Reverse entry

- Invoice chưa sync không được reverse theo policy.
- Invoice đã reverse lần hai trả `ERR_INVOICE_ALREADY_REVERSED`.
- Reverse tạo ID mới và giữ snapshot.
- Không có path update/delete detail đã sync.
- Sửa invoice tạo reverse rồi correction mới.

### Authorization

- Staff được tạo invoice.
- Staff không sửa giá gốc.
- Staff chỉ đọc/tìm customer; không tạo/sửa/xóa customer.
- Staff không được reverse/correction invoice, kể cả khi gửi PIN Admin; TenantAdmin mới được thực hiện reverse.
- TenantAdmin được bulk update đúng tenant.
- Tenant A không truy cập entity tenant B.
- SuperAdmin thao tác global nhưng không vô tình bypass audit.

### Feature flags/i18n

- Tenant override thắng global.
- Default được dùng khi chưa có setting.
- `RequireCustomerPhone` trả error code đúng.
- API không trả business message hardcode.

## 4. Integration test database

Dùng MySQL test container hoặc database test cô lập, không dùng production.

- Global Query Filter tự động thêm tenant condition.
- Insert tenant id lấy từ context, không lấy body.
- Unique `(TenantId, Sku)`, `(TenantId, InvoiceNumber)` hoạt động.
- Concurrent reverse với hai transaction chỉ tạo một reverse.
- `sync_operations` bảo vệ retry cùng operation id.
- Cùng operation id khác payload hash bị reject.
- Transaction rollback nếu insert detail hoặc outbox lỗi.
- Outbox chỉ tồn tại khi invoice commit thành công.
- `Version` ngăn lost update product/settings.

## 5. API contract test

Với mỗi endpoint kiểm tra:

- Schema JSON đúng OpenAPI.
- Field bắt buộc, enum, decimal range và nullable.
- Status code đúng theo tài liệu.
- Error response có `error.code`, không phụ thuộc text.
- JWT thiếu/hết hạn trả 401.
- Role sai trả 403.
- Resource tenant khác trả 404/403 theo policy, không làm lộ tồn tại.
- Request retry idempotent trả cùng kết quả.

## 6. Kịch bản test offline end-to-end

### Scenario A: tạo invoice khi offline

1. Đăng nhập thiết bị đã provision.
2. Tắt network.
3. Tạo invoice hợp lệ.
4. Xác nhận UI đọc invoice từ SQLite.
5. Xác nhận local outbox là `Pending`.
6. Kill app và mở lại.
7. Xác nhận invoice vẫn hiển thị, không duplicate.
8. Bật network.
9. Worker gửi API.
10. Xác nhận server có một invoice, local đổi `Synced`.

### Scenario B: timeout sau server commit

1. Server commit invoice nhưng response bị drop.
2. Client giữ Pending hoặc chuyển retryable.
3. Client retry cùng `ClientOperationId`.
4. API trả kết quả cũ.
5. Database server chỉ có một invoice và một invoice detail set.

### Scenario C: mất mạng giữa batch

1. Tạo 100 invoice local.
2. Sync batch nhỏ.
3. Ngắt mạng sau batch thứ ba.
4. Xác nhận chỉ record chưa ack còn Pending.
5. Bật mạng và retry.
6. Xác nhận đủ 100 invoice, không duplicate.

### Scenario D: revoke device

1. Thiết bị có invoice Pending.
2. Admin revoke device.
3. API trả 401 `ERR_DEVICE_REVOKED`.
4. Mobile dừng worker, xóa SQLite/key/session.
5. User bị đưa về login.

### Scenario E: SignalR broadcast

1. Có hai device cùng tenant và một device tenant khác.
2. Device A sync invoice.
3. Device B nhận `OnDataUpdated` và pull được invoice.
4. Device tenant khác không nhận event.
5. Nếu SignalR down, invoice vẫn tồn tại server; reconnect rồi pull cursor.

### Scenario F: reverse offline

Chính sách nên chốt trước: reverse offline chỉ được tạo local khi có đủ snapshot và server sẽ authorize khi sync, hoặc cấm reverse offline. Test phải bao phủ PIN, invoice đã reverse và conflict khi nhiều thiết bị cùng reverse.

## 7. RabbitMQ integration/chaos test

- Publisher confirm thành công thì outbox chuyển Published.
- Publisher chết sau publish trước mark thì consumer deduplicate.
- Consumer nack transient vào retry queue.
- Message schema lỗi vào DLQ, không retry vô hạn.
- RabbitMQ unavailable không làm mất invoice đã commit.
- Replay DLQ không tạo duplicate.
- Tenant routing key không phát event chéo tenant.

## 8. Load test k6

Kịch bản:

- Login/refresh với token pool.
- 100-1000 thiết bị gửi invoice sync theo batch.
- 5-10% request timeout/retry có chủ đích.
- Pull incremental đồng thời với sync.
- SignalR reconnect sau broker delay.

Đo p50/p95/p99, error rate, duplicate rate, MySQL lock/deadlock, RabbitMQ lag, outbox age và CPU/RAM. Ngưỡng pass phải ghi trong release plan theo cấu hình VPS.

## 9. Regression và release gate

PR gate: build, unit, integration, API contract, static analysis và migration validation.

Staging gate: E2E offline, revoke, reverse, RabbitMQ retry/DLQ, backup restore smoke test.

Production gate: migration review, backup đã thành công, rollback plan, health check, alert và smoke test endpoint login/sync/pull.

## 10. Test data và nguyên tắc

- Dùng tenant/user/device giả lập, không dùng dữ liệu khách thật.
- Test data phải có invoice Sale, Reverse, Correction và trạng thái Rejected.
- Mỗi test dọn dữ liệu hoặc dùng database/container riêng.
- Không log token, PIN, SQLCipher key hoặc payload có PII.
- Flaky test phải được triage, không tăng retry để che lỗi concurrency.
