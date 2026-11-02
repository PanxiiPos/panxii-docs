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

## 9. Dọn dung lượng VPS an toàn

- Đo `df -hT /`, `docker system df`, `journalctl --disk-usage` trước;
  phân biệt dung lượng đĩa với RAM/cache Linux.
- Khi không có build/deploy hoặc cài package đang chạy, có thể dùng
  `docker builder prune --all --force` để xóa build cache không dùng và
  `apt-get clean` để xóa gói đã tải. Build sau có thể chậm hơn vì mất cache.
- Không dùng `docker system prune --all --volumes`; không xóa trực tiếp
  thư mục Docker/containerd, DB/broker volumes, backup, `.env`, TLS/key,
  release source hoặc image rollback. Không truncate log/evidence pilot để
  giải phóng dung lượng khi đang nghiệm thu.
- Sau dọn, đối chiếu image IDs, volume names và container IDs trước/sau;
  chạy host monitor, xác minh PID và mẫu mới của soak. Không cần restart app.

Ngày 2026-11-02: dọn build cache và APT cache, thu hồi thực tế **6,72 GiB**;
đĩa từ **34% xuống 20%**, còn **37,66 GiB** khả dụng. Giữ nguyên 23 images
(bao gồm rollback), 7 containers đang chạy và 3 volumes; soak giữ PID/mẫu mới,
host monitor pass. Evidence root-only trên VPS: hồ sơ vận hành nội bộ.
Không cài lịch tự động prune hoặc thay đổi chính sách backup.
