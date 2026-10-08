# OPERATIONS RUNBOOK

## 1. Health check

- `/health/live`: process còn sống.
- `/health/ready`: MySQL/RabbitMQ sẵn sàng.
- API health fail không đồng nghĩa database mất dữ liệu; kiểm tra metrics/log trước khi restart.

## 2. Xử lý outbox tăng cao

1. Kiểm tra queue depth, publisher error và MySQL latency.
2. Kiểm tra worker còn chạy và không bị deadlock.
3. Không xóa outbox pending để giảm queue.
4. Scale worker hoặc sửa dependency, sau đó replay theo batch.
5. Kiểm tra duplicate rate và event tenant routing.

## 3. Xử lý DLQ

1. Xem `messageId`, `eventType`, `tenantId`, error code và retry count.
2. Phân loại transient, schema hoặc business error.
3. Sửa nguyên nhân trước khi replay.
4. Replay có phê duyệt, consumer idempotent.
5. Ghi incident và cập nhật error handling nếu cần.

## 4. MySQL incident

- Kiểm tra connection pool, slow query, lock/deadlock, disk và replica/backup.
- Không kill transaction invoice đang xử lý khi chưa hiểu trạng thái idempotency.
- Sau restore chạy tenant count, invoice count, outbox consistency và sync smoke test.

## 5. SignalR incident

SignalR mất không được coi là mất invoice. Kiểm tra invoice server, sau đó client reconnect và gọi `/sync/pull` từ cursor cuối.

## 6. Device revoke

Admin revoke device; mobile nhận `401 ERR_DEVICE_REVOKED`, dừng worker, xóa SQLite/key/token và login lại. Pending data bị xóa theo chính sách bảo mật hiện hành.

## 7. Certificate/secret

- Renew certificate trước hạn và test staging.
- Rotate JWT/RabbitMQ/DB secret theo runbook.
- Không ghi secret vào log hoặc chat incident.

## 8. Incident severity

| Mức | Ví dụ | Phản ứng |
|---|---|---|
| P1 | Mất dữ liệu/tenant chéo | Chặn traffic, bảo toàn log, incident commander |
| P2 | Sync lỗi diện rộng/DLQ tăng | Giảm rollout, xử lý worker/broker |
| P3 | Một feature lỗi | Tạo hotfix/rollback có kiểm soát |
| P4 | UI/metric nhỏ | Đưa vào backlog |
