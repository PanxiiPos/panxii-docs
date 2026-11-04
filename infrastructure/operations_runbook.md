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
host monitor pass. Evidence root-only trên VPS: `/opt/panxii/cleanup-20261102.json`.
Không cài lịch tự động prune hoặc thay đổi chính sách backup.

## 10. DNS host chậm trong pilot

Soak `20261104-042043` dừng lúc 12:12:33 ngày 04/11/2026 UTC+7 vì read
p95 1344,62 ms > 500 ms. Các catalog request trong cửa sổ đó vẫn HTTP 200,
Nginx request time 25-40 ms. Probe sau incident cho thấy DNS lookup trực tiếp
thường 260-391 ms; không có phase timing của request lỗi để khẳng định chính
xác toàn bộ 1344,62 ms là DNS. Không nới ngưỡng hoặc bỏ DNS/TLS khỏi phép đo.

Owner duyệt dùng cache TTL của `systemd-resolved` đã có sẵn trên VPS:

- `/etc/resolv.conf` trỏ tới `/run/systemd/resolve/stub-resolv.conf`.
- `/etc/systemd/resolved.conf.d/60-panxii-cache.conf`: `[Resolve]`,
  `DNS=8.8.4.4 8.8.8.8`, `FallbackDNS=`, `Domains=~.`, `Cache=yes`,
  `DNSStubListener=yes`. Directory phải mode 755, file 644 để service đọc được.
- `systemctl enable --now systemd-resolved`; kiểm tra `resolvectl status`
  thực sự có global DNS/domain trên, không chỉ kiểm tra file đã ghi.
- Dùng `resolvectl statistics` kiểm chứng cache hits; đo cả cold lookup sau
  `resolvectl flush-caches`, lookup lặp lại và HTTPS sau 70 giây idle/TTL expiry.
  Không ghi `/etc/hosts`, ghim IP, bỏ TLS verification hoặc ép TTL dài hơn DNS.
- Không cần `netplan apply`, restart Docker hay API/Worker. Container ID/image/
  mount/start time phải giữ nguyên. Cache host không chứng minh DNS của điện
  thoại hoặc Docker containers đã được thay đổi/kiểm chứng.

Backup resolver cũ: `/opt/panxii/resolv.conf.before-dns-cache-20261104`.
Rollback khi cần: thay symlink `/etc/resolv.conf` bằng bản backup đó, bỏ đúng
drop-in `60-panxii-cache.conf`, rồi disable/stop `systemd-resolved` (trạng thái
trước thay đổi là disabled/inactive). Xác minh `getent ahosts` và HTTPS sau
rollback; không xóa TLS, data hoặc các config không liên quan.
