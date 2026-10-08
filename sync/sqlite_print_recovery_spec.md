# SQLITE, SQLCIPHER VÀ PRINT RECOVERY SPEC

## 1. SQLite local

- SQLite/SQLCipher là source of truth cho UI offline, không phải source of truth kế toán.
- Bật WAL theo platform support và kiểm tra bằng integration test.
- Invoice và local outbox commit trong cùng transaction.
- `local_change_cursor` chỉ cập nhật sau khi upsert batch thành công.
- Mỗi local schema có `schemaVersion` và migration test.
- SQLCipher key lấy từ secure storage, không log và không lưu cùng database.

## 2. Recovery policy

| Lỗi | Hành động |
|---|---|
| Database busy | Backoff, retry giới hạn |
| Disk full | Rollback transaction, báo lỗi, không mark invoice success |
| WAL recovery | Mở lại và integrity check |
| SQLCipher key invalid | Force logout/recovery flow, không loop |
| Corrupt database | Đóng connection, backup file lỗi nếu policy cho phép, recreate DB, full pull |
| Device revoked | Dừng worker, xóa DB/key/token |

Không tự động xóa database khi lỗi tạm thời. Chỉ xóa sau khi phân loại corruption/revoke theo policy.

## 3. Printer out-of-band

Invoice commit và printer là hai state machine độc lập.

```text
Invoice: Pending -> Synced
PrintJob: Pending -> Printing -> Completed | Failed
```

- Printer timeout không rollback invoice/payment.
- Retry/reprint dùng `PrintJobId`.
- UI hiển thị invoice success và print failure riêng.
- Print payload lấy từ invoice snapshot, không đọc giá hiện tại.
- Print worker có backoff, attempt limit và error code.

## 4. Acceptance tests

- Kill app ở mọi điểm giữa invoice/outbox transaction.
- Hết disk trong commit.
- Mất điện khi WAL đang ghi.
- Sai SQLCipher key.
- Reopen sau corruption.
- Printer offline/timeout/mất giữa job.
- Retry print không tạo invoice thứ hai.
