# Kế hoạch 6 công việc trước production

- Ngày lập: 2026-11-01.
- Phạm vi: cửa hàng gia đình, offline-first, chấp nhận backend gián đoạn;
  ưu tiên dịch vụ miễn phí và thiết bị hiện có.
- Trạng thái: **kế hoạch, chưa triển khai các công việc bên dưới**.
- Không tự nâng gói trả phí, dùng credit trial như miễn phí lâu dài hoặc
  thay đổi dữ liệu development/production để chuẩn bị release.

## 1. Điểm xuất phát và nguyên tắc

Đã có bằng chứng local: backend resilience 22/22 trên hai stack; Android
development 3+8 checkpoint; 8 invoice không trùng; backup development restore
khớp 18 bảng. Đây không phải bằng chứng hạ tầng production đang hoạt động.
Xem [ma trận kiểm thử](../testing/risk_based_testing_spec.md).

Ngày 2026-11-02 đã dựng VPS private: API/Worker Production, MySQL/RabbitMQ,
17 migrations và duy nhất một SuperAdmin; health, login/platform/logout,
bootstrap chống ghi đè và restart API/Worker đã pass. Các bảng nghiệp vụ trống.
Secrets chỉ trên VPS `.env` quyền 600; SSH key-only, API chỉ loopback.
Chưa có domain/HTTPS, backup ngoài máy, pipeline deploy, signing release
hoặc Shorebird patch/rollback; **chưa GO production**. Oracle Always Free đã
bị loại, VPS thuê thay thế. Source đang ở branch feature; không đồng nghĩa
đã merge. Xem [trạng thái VPS](../infrastructure/internal_free_deployment.md).

Chuẩn bị script/config trên development. Dựng VM dự kiến production nhưng
**chưa nhận giao dịch thật**, dùng tenant test riêng để nghiệm thu. Chỉ chuyển
sang vận hành sau quyết định GO. Sau GO, không chạy chaos, stress hoặc restore
ghi đè trên hệ thống cửa hàng.

Kế hoạch theo [release management](release_management.md),
[hướng dẫn triển khai](../infrastructure/internal_free_deployment.md) và
[quy trình PR](../development_workflow.md). Khi có xung đột, theo thứ tự ưu tiên
của [documentation hub](../README.md); không tự đổi business rule.

## 2. Sáu công việc

### 1) Đóng gói và dựng hạ tầng production

**Nơi làm:** development rồi VM chưa khai thác.

- Xác nhận đã thuê VPS đúng cấu hình và trong ngân sách được chủ dự án duyệt;
  không tự nâng gói/cấu hình khi chưa được duyệt.
- Tạo Compose production riêng: API, Worker, MySQL, RabbitMQ, Nginx.
  API/Worker dùng Production, credentials riêng, volumes bền vững và restart
  policy; giới hạn tài nguyên/log theo capacity đo được.
- Build API/Worker đúng kiến trúc VM (VPS thường là x64; nếu ARM64 thì kiểm tra
  image hạ tầng có hỗ trợ). Pin image ứng dụng theo SHA và hạ tầng theo
  version/digest.
- Thiết lập SSH key/host verification, đồng bộ giờ, OS firewall và firewall của
  nhà cung cấp VPS (nếu có).
  Chỉ cho phép cổng quản trị từ nguồn kiểm soát; không publish DB/broker/API.
- Review migrations, provisioning tenant/admin ban đầu và quyền migration
  riêng; không chạy seed demo, dùng mật khẩu mặc định hay migration phá hủy
  tự động khi app khởi động.
- Không dùng lại volume development; chưa xóa dữ liệu demo hiện có. Khởi tạo
  production sạch là thao tác riêng, phải xác nhận đúng đích.

**Bàn giao:** Compose, quy trình bootstrap/provisioning, image SHA/digest,
inventory VM/volume/ports và checklist quota.

**Đạt khi:** stack chạy trên đúng kiến trúc, DB/broker không public; restart VM
giữ dữ liệu fixture và services trở lại; health/login/sync smoke pass.
Chỉ diễn tập restart trước khi có giao dịch thật.

### 2) Backup tự động ngoài VM và phục hồi độc lập

**Phụ thuộc:** công việc 1 để nghiệm thu trên VM; script có thể viết trước.

- Chốt nơi lưu ngoài VM, dung lượng/retention và kênh cảnh báo trong quota
  hiện có. Không coi Docker volume hoặc bản sao cùng VM là backup.
  **Đã chốt 2026-11-02:** lưu Google Drive qua Rclone; xem lịch/retention bên dưới.
- Viết job dump nhất quán, nén, mã hóa xác thực; fail-closed nếu dump thiếu,
  upload lỗi hoặc hết quota. Chống hai job chạy chồng nhau.
- Chọn cách giữ và phục hồi key độc lập; không phụ thuộc duy nhất tài khoản
  Windows/DPAPI đang dùng ở test development.
- Lập lịch tự động; xác minh artifact tồn tại, có hash, tải về được và không
  chứa secrets plaintext. Retention chỉ xóa bản cũ sau khi bản mới đã xác minh.
- Restore backup tải từ storage vào stack tách biệt, dùng key phục hồi.
  Đối chiếu integrity/fingerprint và login/sync/pull/reverse/idempotency.
- Đo RPO/RTO và diễn tập mất VM bằng môi trường mới; không restore đè nguồn.

**Đã chốt ngày 2026-11-02, chưa phải SLA:** backup 3 lần/ngày lúc 06:00, 12:00
và 21:00 (UTC+7) lên Google Drive qua Rclone; chỉ giữ 6 bản mới nhất (khoảng
2 ngày lịch sử), bản cũ hơn tự xóa, và chỉ xóa sau khi bản mới đã xác minh.
Bổ sung backup thủ công trước deploy/migration, lưu riêng ngoài chu kỳ tự xóa.
Khoảng cách lớn nhất giữa hai lượt là 9 giờ nên RPO thực tế có thể mất tới
~9 giờ dữ liệu server — giới hạn do chủ dự án chấp nhận. Offline-first không
tự bảo đảm mọi dữ liệu server có thể lấy lại từ điện thoại. RTO chốt sau drill
thực tế.

**Đạt khi:** ít nhất hai lịch tự động liên tiếp thành công; lỗi dump/upload
tạo cảnh báo đã nhận được; restore từ storage/key độc lập pass và ghi RPO/RTO.

### 3) Nginx, HTTPS và đường truy cập mobile

**Phụ thuộc:** công việc 1; cần hostname và DNS được kiểm soát.

- Chọn subdomain/DDNS miễn phí phù hợp nếu chưa mua domain; kiểm tra điều khoản,
  quyền DNS, độ ổn định và khả năng cấp/gia hạn chứng chỉ.
- Cấu hình Nginx reverse proxy, DNS service khi recreate API và WebSocket cho
  SignalR. HTTPS dùng certificate hợp lệ; không bỏ xác minh TLS trên mobile.
- Wire forwarded headers đúng trusted proxy; kiểm tra HTTPS redirection không
  loop, địa chỉ client dùng cho rate limit và hạn mức request.
- Chặn origin services bằng Docker/OS/firewall nhà cung cấp, không chỉ đổi URL.
  Nginx che topology/service port, **không bảo đảm giấu IP máy chủ hoặc đường
  API khỏi người dùng app**. Auth và authorization vẫn bắt buộc.
- Kiểm tra public Swagger/health theo policy vận hành, redaction token query
  SignalR trong access log và giới hạn log.
- Tự gia hạn chứng chỉ; chạy renewal dry-run và kiểm tra alert hết hạn.

**Đạt khi:** thiết bị cửa hàng login/sync/pull/SignalR qua HTTPS pass; chứng chỉ
đúng hostname; HTTP redirect đúng; truy cập ngoài vào 3306/5672/15672/API bị chặn;
renewal dry-run pass.

### 4) APK ký release, cập nhật và Shorebird

**Phụ thuộc:** công việc 3 để build/nghiệm thu API URL thật.

- Thiết lập release keystore riêng, backup key an toàn ngoài Git; release build
  thiếu signing secrets phải fail, không fallback debug key.
- Pin Flutter/Shorebird tương thích; build số phiên bản mới với API HTTPS,
  HOME_PREVIEW tắt; đối chiếu chữ ký và hash artifact.
- Publish APK nền Shorebird qua kênh tải HTTPS; cấu hình native update policy
  với minimum/latest/download URL đã kiểm thử.
- Trên thiết bị pilot: cài APK, offline -> restart -> sync, expired session,
  lỗi cập nhật và giữ dữ liệu chờ đồng bộ.
- Test Dart-only patch vào đúng release/track pilot, tải -> áp dụng sau restart,
  và rollback patch. Assets/native thay đổi phải tạo APK mới.
- Chuyển từ debug signature sang release có thể không cài đè được:
  kiểm tra pending invoice/outbox và kế hoạch chuyển dữ liệu trước khi uninstall.
  Không tự xóa dữ liệu app để vượt lỗi chữ ký.

**Đạt khi:** chữ ký release đúng, key phục hồi được, APK/API HTTPS hoạt động trên
thiết bị cửa hàng; native update và patch/apply/rollback có bằng chứng.
Nếu chưa dùng OTA ngày đầu, chỉ được hoãn nghiệm thu patch khi chủ cửa hàng
xác nhận OTA ngoài phạm vi go-live và có đường cập nhật APK đầy đủ đã đạt.

### 5) CI/CD, monitoring và rollback vận hành

**Phụ thuộc:** 1, 2, 3 để nghiệm thu deploy; chuẩn bị workflow song song.

- Tạo gate PR/main: build, focused/full tests theo phạm vi, contract/migration,
  dependency/secret scans. AI review hiện có không thay thế các gate này.
- Build/push image SHA đúng kiến trúc; pin actions và giới hạn artifact/quota.
  Secrets quyền tối thiểu; không đưa backup/key vào CI artifact.
- Sau merge main đã được duyệt: serialize deploy, verify SSH host, backup
  trước migration, apply migration đã review, deploy API/Worker giữ volumes.
- Health/smoke fail phải báo lỗi và rollback image đã biết hoạt động;
  không tự migration down hoặc restore đè dữ liệu.
- Giám sát nhẹ trước, tránh thêm stack nặng không cần thiết:
  health/5xx, disk/RAM, backup age, certificate expiry, outbox age,
  RabbitMQ depth/unacked/DLQ và worker recovery.
- Gửi alert vào kênh chủ cửa hàng thực sự nhận được; inject lỗi test để xác
  minh alert. Viết lệnh/runbook cho restore, replay DLQ có duyệt và rollback.
- Giữ tối thiểu một image/API/Worker và APK trước để phục hồi.

**Đạt khi:** CI fail chặn deploy; một lượt deploy và rollback staging pass;
alert backup/service/disk hoặc ngưỡng mô phỏng được nhận; runbook có người
chịu trách nhiệm và không cần secret trong chat.
Auto deploy main không được bật trước nghiệm thu bootstrap/pilot.

### 6) Nghiệm thu capacity, soak, pilot và quyết định GO/NO-GO

**Phụ thuộc:** 1-5; chuẩn bị workload/script trước, chạy khi stack đã sẵn sàng.

- Chốt số điện thoại đồng thời, invoice/ngày, backlog sau mất mạng, catalog
  và tốc độ tăng dữ liệu. Không suy đoán capacity từ 16 retry cùng UUID.
- Tạo workload tenant test: tải bình thường, nhiều thiết bị replay backlog,
  retry trùng và SignalR reconnect. Đếm transaction đã commit, không chỉ 202.
- Load test trên đúng cấu hình VM; đo p50/p95/p99, throughput, 5xx,
  publish confirm, thời gian acceptance -> Completed -> mobile Synced,
  CPU/RAM/disk, queue/outbox age.
- Stress tăng tải có giới hạn trên VM chưa khai thác để tìm ngưỡng; sau giảm
  tải phải drain backlog và không mất/trùng invoice. Không dùng stress để tạo
  tải giả tránh chính sách thu hồi VM.
- Soak có giới hạn thời gian, đề xuất 24 giờ với tải đại diện; đo baseline/end
  và xu hướng RAM, connections, logs, disk, queue. Không để tool chạy vô hạn.
- Lặp resilience gate đúng build, contract/tenant tests, migration/restore
  drill và E2E thiết bị. Không tái dùng runner development để kill production.
- Nếu in là chức năng ngày đầu: thử Canon/QR/A4, timeout/reprint không trùng
  và bridge tự chạy sau PC restart. Nếu không, xác nhận tắt/ngoài phạm vi.
- Pilot giới hạn trước rollout. Pilot nhận dữ liệu thật chỉ sau các gate bảo
  vệ dữ liệu/signing/HTTPS đã đạt; không chạy chaos/stress trong pilot đó.

**Ngưỡng đề xuất cần duyệt trước test:** tham khảo
[SLO baseline](../infrastructure/slo_observability.md), không tự coi là SLA:
read p95 <= 500 ms, pull p95 <= 1,5 s; báo riêng POST acceptance và thời gian
sync end-to-end (mục tiêu p95 <= 1,5 s khi dependency healthy).
Backlog phục hồi phải có deadline chốt theo kích thước workload; sau drain
không có DLQ không xử lý, missing hoặc duplicate transaction.
Soak không có crash/OOM, queue tích tụ dai dẳng hoặc tăng tài nguyên không
giải thích được; mọi mức cap phải nằm trong quota đã xác nhận.

**Đạt khi:** report ghi workload/build/VM/ngưỡng/results; chủ cửa hàng xác nhận
capacity phù hợp, backup/rollback có thể thực hiện và checklist GO bên dưới.
Stress cực hạn không thay thế load/soak hoặc nghiệm thu tính đúng dữ liệu.

## 3. Thứ tự triển khai

```text
Development: chuẩn bị config/script 1, 2, 3, 4, 5 và workload 6
VM chưa khai thác: 1 -> [2 + 3] -> [4 + 5] -> 6 -> GO
```

Nghiệm thu 5 cần 2 và 3; nghiệm thu 4 cần 3. Pilot của 6 cần tất cả gate bảo
vệ dữ liệu trước đó. Không dùng thứ tự song song để bỏ prerequisite.
Không đặt ngày go-live cố định trước khi xác nhận VPS/hostname/storage.

## 4. Những quyết định cần chủ cửa hàng chốt

1. Nhà cung cấp VPS và nơi lưu backup **đã chốt**: thuevpsgiare.vn gói VPS SSD C4; Google Drive qua Rclone. Còn phải chốt hostname/DDNS.
2. RPO/retention **đã chốt** (3 lượt/ngày, giữ 6 bản mới nhất, chấp nhận mất tới ~9 giờ). Còn phải chốt RTO sau drill và người giữ recovery key.
3. Số thiết bị/tải/backlog cần nghiệm thu; ngưỡng latency và soak duration.
4. OTA và in Canon có bắt buộc ngày đầu không; thiết bị pilot/kênh tải APK.
5. Kênh nhận cảnh báo và người quyết định rollback/GO.

Chỉ hỏi khi đến công việc liên quan; không yêu cầu người dùng gửi credentials
vào chat. Tạo secrets qua nơi lưu phù hợp và giữ mọi tài nguyên trong quota.

## 5. Checklist GO/NO-GO

- [ ] Hạ tầng Production/kiến trúc/quota/volumes/provisioning pass, không demo secret.
- [ ] Backup tự động ngoài VM và restore bằng recovery key độc lập pass.
- [ ] HTTPS/renewal/private services/SignalR thiết bị thật pass.
- [ ] APK release ký đúng key; đường cập nhật và OTA theo phạm vi đã chốt pass.
- [ ] CI gates, monitoring/alert, deploy/rollback và runbook pass.
- [ ] Load/soak/correctness/pilot đạt ngưỡng, in nếu trong phạm vi pass.
- [ ] Owner duyệt kết quả và release notes; không còn blocker dữ liệu/tenant.

Một gate bắt buộc fail hoặc thiếu evidence => **NO-GO**. Chấp nhận downtime
không đồng nghĩa chấp nhận mất dữ liệu, thiếu backup hoặc bỏ TLS.
Lưu evidence theo release SHA, không đưa secrets/PII/backup plaintext vào Git.
