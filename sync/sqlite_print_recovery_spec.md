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
PrintJob (PC): accepted -> printing -> submitted | uncertain | failed
```

- Printer timeout không rollback invoice/payment.
- Retry/reprint dùng `PrintJobId`.
- UI hiển thị invoice success và print failure riêng.
- Khi màn hình in đang mở, app tự kiểm tra cùng UUID/nội dung sau mỗi 2 giây
  nếu trạng thái là `accepted`/`printing`, tối đa 30 lần. Terminal hiển thị
  dialog và trạng thái trên màn hình; `submitted` chỉ xác nhận gửi Windows.
  Hết lượt hoặc lỗi mạng dừng tự kiểm tra, báo rõ và cho kiểm tra thủ công
  cùng lệnh; đóng màn hình hủy timer, không hủy job PC hoặc tự in thêm.
- Print payload lấy từ invoice snapshot, không đọc giá hiện tại.
- Bridge persist UUID/hash trước khi worker claim; gửi lại cùng UUID/nội dung chỉ đọc trạng thái. Không retry tự động job đã bắt đầu spool. Crash tại printing chuyển uncertain; chỉ in thêm bản mới sau xác nhận và kiểm tra queue/giấy. Submitted không phải giấy đã in.
- Điện thoại lưu journal theo hash tài liệu trong secure storage tenant/user trước network; cùng tài liệu giữ UUID qua restart và in tài liệu khác xen giữa. Đổi ảnh (mẫu/ngôn ngữ/cấu hình) là tài liệu mới; phải kiểm tra queue nếu đã có lần in trước.
- Ảnh chờ in được giữ trong SQLite PC rồi xóa khi terminal; metadata dedup được giữ. Pairing token/cert chỉ local; xem [vận hành bridge](../../tools/windows-print-bridge/README.md).

## 4. Acceptance tests

- Kill app ở mọi điểm giữa invoice/outbox transaction.
- Hết disk trong commit.
- Mất điện khi WAL đang ghi.
- Sai SQLCipher key.
- Reopen sau corruption.
- Printer offline/timeout/mất giữa job.
- Retry print không tạo invoice thứ hai.
