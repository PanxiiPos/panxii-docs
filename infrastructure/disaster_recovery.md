# DISASTER RECOVERY VÀ BACKUP POLICY

## 1. Mục tiêu cần chốt

- **RPO:** lượng dữ liệu tối đa chấp nhận mất.
- **RTO:** thời gian tối đa để phục hồi dịch vụ.

Giá trị chính thức phải được chốt theo gói SaaS; tài liệu này không tự đặt SLA khi chưa có capacity plan.

## 2. Backup

- MySQL backup hằng ngày hoặc theo RPO.
- Backup mã hóa, lưu ngoài VPS và có retention nhiều phiên bản.
- Lưu cấu hình deployment, migration metadata và secret recovery procedure riêng.
- RabbitMQ event không thay thế backup MySQL.

## 3. Restore drill

Ít nhất định kỳ:

1. Tạo môi trường restore cô lập.
2. Restore backup.
3. Chạy integrity check tenant/invoice/outbox.
4. Chạy API login, sync, pull và reverse smoke test.
5. Đo thời gian restore.
6. Ghi kết quả và khắc phục gap.

## 4. Mất VPS

1. Chặn DNS/traffic nếu cần.
2. Provision VPS mới, harden firewall/SSH.
3. Deploy image đã pin version.
4. Restore MySQL và volumes cần thiết.
5. Cấu hình secrets/certificate.
6. Kiểm tra health, tenant isolation và sync.
7. Chuyển DNS sau khi smoke test pass.

## 5. Lộ secret

1. Revoke/rotate secret bị lộ ngay.
2. Rotate JWT signing key theo compatibility plan.
3. Revoke device/session nghi ngờ.
4. Kiểm tra audit/log truy cập.
5. Đánh giá PII và thông báo theo chính sách pháp lý.

## 6. Data integrity

Không dùng restore để giải quyết duplicate invoice nếu chưa kiểm tra `ClientOperationId` và `sync_operations`. Mọi sửa dữ liệu giao dịch phải qua reverse entry hoặc quy trình được phê duyệt.
