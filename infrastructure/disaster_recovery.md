# DISASTER RECOVERY VÀ BACKUP POLICY

## 1. Mục tiêu cần chốt

- **RPO:** lượng dữ liệu tối đa chấp nhận mất.
- **RTO:** thời gian tối đa để phục hồi dịch vụ.

Giá trị chính thức phải được chốt theo gói SaaS; tài liệu này không tự đặt SLA khi chưa có capacity plan.

## 2. Backup

- Chính sách nội bộ đã chốt ngày 2026-11-02: dump MySQL mã hóa, upload Google
  Drive bằng Rclone lúc 06:00, 12:00 và 21:00 (UTC+7); giữ 6 bản mới nhất, bản
  cũ chỉ xóa sau khi bản mới đã xác minh. RPO thực tế tối đa ~9 giờ; chi tiết
  và giới hạn tại [hướng dẫn triển khai](internal_free_deployment.md#31-backup-mysql-lên-google-drive-bằng-rclone).
- Backup mã hóa, lưu ngoài VPS và có retention nhiều phiên bản theo chính sách trên.
- Lưu cấu hình deployment, migration metadata và secret recovery procedure riêng.
- RabbitMQ event không thay thế backup MySQL.

### 2.1. Triển khai VPS ngày 2026-11-02

- Script production tại [backend deploy](../../backend/deploy/README.md):
  MySQL transactional dump + gzip, Rclone crypt mã hóa tên/nội dung trước
  Google Drive, download/decrypt compare và marker readback trước retention.
- Systemd timer dùng Asia/Ho_Chi_Minh đúng 06/12/21; giữ 6 bản scheduled đã
  xác minh, xóa vĩnh viễn bản cũ thay vì đưa vào Drive Trash. Manual trước
  deploy/OS update nằm riêng, không bị scheduled retention xóa.
- `.env` root:root 600 giữ OAuth/crypt settings. File khôi phục chỉ chứa crypt
  settings được xuất ngoài VPS với scrypt/AES-GCM; mật khẩu do owner nhập kín,
  không phụ thuộc Windows DPAPI. Giữ file và mật khẩu độc lập với VPS.
- Manual backup và service-triggered smoke đã thành công. Tải bản từ Drive,
  restore MySQL disposable `--network none`, inventory và data fingerprint
  của cả 18 bảng khớp; khoảng 33 giây chỉ là thời gian drill nhỏ, không phải
  RTO đầy đủ hay cam kết workload thật.
- Chưa có hai lịch tự động liên tiếp thực sự; chưa diễn tập mất VPS, recovery
  từ file/passphrase trên môi trường mới hoặc login/sync/pull/reverse bằng
  dữ liệu restore. Không đánh dấu toàn gate backup/DR đã đạt.
- Backup failure gửi cảnh báo Discord; fault injection đã được owner xác nhận
  nhận. Host monitor cảnh báo bản scheduled quá 10 giờ; đây không phải
  availability monitor bên ngoài hoặc thay đổi RPO ~9 giờ.

## 3. Restore drill

Ít nhất định kỳ:

1. Tạo môi trường restore cô lập.
2. Restore backup.
3. Chạy integrity check tenant/invoice/outbox.
4. Chạy API login, sync, pull và reverse smoke test.
5. Đo thời gian restore.
6. Ghi kết quả và khắc phục gap.

### 3.1. Gate tự động trên môi trường cô lập

Từ thư mục `backend`, chạy:

```powershell
.\scripts\test-resilience.ps1
```

Runner cần .NET 8 SDK, Docker Linux engine và Compose v2 trở lên. Runner tự
tạo project `panxii-resilience-<UUID>`, credential giả, port loopback động,
hai database/volume riêng và broker riêng; không nhận URL hoặc credential
production. Nó cố ý SIGKILL service test và xóa **chỉ volume của project test**
khi kết thúc. Không thay runner để trỏ vào stack đang sử dụng thật.

Gate kiểm tra dump MySQL thực tế, nén và mã hóa AES-GCM, đọc lại artifact,
restore sang MySQL riêng và so fingerprint toàn bộ bảng (không chỉ row count).
API/Worker restore dùng vhost riêng; login, replay operation, sync/pull và reverse phải
hoạt động mà không ghi dữ liệu thử vào database nguồn. Backup hỏng, sai key,
truncated, dump không hoàn tất hoặc lệnh dump thất bại không được báo thành công.
Key chỉ dùng trong test; đây **không phải** script backup production.

Các test còn kiểm tra crash giữa hai lần SaveChanges trước commit, transaction
rollback, retry trùng, broker/worker/API/Nginx restart, DLQ và SignalR/pull.
Chi tiết và giới hạn bằng chứng tại
[risk_based_testing_spec.md](../testing/risk_based_testing_spec.md#10-gate-resilience-trước-triển-khai).

TRX được lưu tại `backend/artifacts/resilience/<project>/resilience.trx`.
`manifest.json` cùng thư mục ghi commit, working tree dirty, SHA-256 source,
thời gian chạy và cleanup; source đổi trong lúc test thì gate fail.
Runner trả exit code khác 0 nếu test fail, skip, thiếu report, số test không đủ
hoặc cleanup lỗi. Không retry nguyên test suite tự động để che lỗi.

**Chưa được chứng minh bởi gate local:** upload/download tới storage ngoài VM
thực tế, cron
production, retention/alert, HTTPS/certificate renewal, recovery từ mất cả VM,
RPO/RTO của dữ liệu thật, SQLCipher trên thiết bị và Shorebird OTA. Những mục
này vẫn là điều kiện release riêng; test local pass không tự cấp phép deploy.

### 3.2. Snapshot dữ liệu development hiện có trước fault test mobile

Runner `backend/scripts/test-development-mobile.ps1 -AllowDevelopmentInterruption`
tạo backup **database development hiện có**, không phải database tổng hợp của
mục 3.1. API/Worker dừng trong lúc snapshot; dump được nén và mã hóa AES-GCM,
đọc lại rồi restore vào MySQL disposable. Fingerprint toàn bộ bảng phải khớp
trước khi deploy lại API/Worker. Không restore đè database nguồn.

Artifact nằm ở `backend/artifacts/development-mobile/<run>/database.backup`;
`backup-key.dpapi` giữ key được bảo vệ bằng Windows DPAPI cho tài khoản chạy
runner, `backup-restore.json` ghi hash artifact và số bảng được đối chiếu.
Đây là bản sao local có thể đọc lại bằng tài khoản Windows đó và helper
`RecoveryBackup` trong test; không phải cron/retention/off-VM backup production.
Không coi việc copy ciphertext và DPAPI key sang VM Linux là key recovery.
Xem [phạm vi và bằng chứng mobile](../testing/risk_based_testing_spec.md#11-gate-mobile-trên-development-đang-chạy).

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
