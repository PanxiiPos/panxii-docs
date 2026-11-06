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
host monitor pass. Evidence chi tiết được lưu trong hồ sơ vận hành nội bộ.
Không cài lịch tự động prune hoặc thay đổi chính sách backup.

Ngày 2026-11-05 lúc 00:03: dọn Docker build cache không dùng và APT cache,
giữ release lock, không có build/package installation đang chạy. Thu hồi
thực tế **5,04 GiB**; đĩa từ **32% xuống 21%**, còn **36,84 GiB** khả dụng.
Đối chiếu giữ nguyên đủ 35 image IDs (gồm rollback), 7 container IDs,
image/start time/mounts và 3 volumes. Không restart service, không xóa backup,
APK, source release, log hoặc dữ liệu nghiệp vụ. Build cache còn 0; host
monitor và public health pass. Evidence chi tiết được lưu trong hồ sơ vận hành nội bộ.
Không bật lịch dọn tự động; lần build sau phải tạo lại cache.

## 10. Sự cố hiệu năng DNS

- Dùng monitoring được phê duyệt để phân biệt thời gian phân giải tên,
  thiết lập kết nối TLS và thời gian xử lý ứng dụng.
- Mọi thay đổi resolver hoặc DNS cache phải được owner phê duyệt, tôn trọng
  TTL và giữ nguyên xác minh TLS. Kiểm tra cả trường hợp cache lạnh, cache ấm
  và HTTPS trước khi kết luận.
- Lưu cấu hình cụ thể và hướng dẫn rollback trong hồ sơ vận hành riêng.
  Không ghim địa chỉ IP hoặc bỏ qua xác minh TLS.

## 11. Keepalive HTTPS và pilot

- Thay đổi keepalive proxy/client cần được phê duyệt và kiểm tra trên môi
  trường phù hợp trước khi áp dụng. Giữ timeout client tương thích với proxy.
- Xác minh kết nối tái sử dụng và kết nối mới; kết nối mới vẫn phải xác minh
  TLS. Sao lưu cấu hình và giữ hướng dẫn rollback trong hồ sơ vận hành riêng.
- Không tự replay workload đã lỗi. Ghi lại kết quả kiểm thử và giới hạn của
  bằng chứng; không xem kiểm tra giới hạn là nghiệm thu dài hạn.
