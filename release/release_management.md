# RELEASE MANAGEMENT

Kế hoạch triển khai trước go-live:
[6 công việc production readiness](production_readiness_plan.md).
Đây là kế hoạch và điều kiện nghiệm thu, không phải xác nhận đã triển khai.

## Bằng chứng release production cập nhật ngày 2026-11-05 — chưa GO

### Mobile native production 0.1.6+8

- OTA stable patch `1` (`780257`), source `9a6aac6`, phát hành ngày
  05/11/2026 cho `0.1.6+8`, Android arm64, Flutter `3.47.8`.
  Chỉ dòng đầu header trái A5 in đậm, không tăng cỡ chữ; địa chỉ/SĐT và
  header phải giữ nguyên. Preview và PNG gửi in dùng chung renderer,
  regression kiểm tra cả có/không QR ngân hàng. 49 test liên quan pass,
  full Flutter analyze sạch; native/assets compatibility pass không override.
  SHA-256 patch:
  `cfea15e60509cf80cc70ee612782e2597a753319d357e4fa0ad16e31d3405597`.
- Owner xác nhận nhận OTA và app tự mở lại trên Samsung SM-A336E.
  ADB chỉ quan sát: PID `19372` -> `21287`, RestartActivity, patch signature
  valid, active path `patches/1/dlc.vmcode`, successful launch và
  `current_patch_number: Some(1)`. Không cài APK, force-stop hoặc gửi lệnh
  mở lại app trong lần test OTA này. Public Shorebird check xác nhận patch 1
  available trên stable, không có rolled-back patch. Native vẫn `0.1.6+8`,
  không đổi API policy/backend/tool in. Chưa nghiệm thu giấy in của patch.
- Owner yêu cầu phát hành APK nền mới để hỗ trợ tự restart sau OTA.
  Shorebird native release `0.1.6+8` đã publish bằng Flutter pin `3.47.8`,
  chữ ký patch do Shorebird quản lý, `auto_update: false`.
- [APK production 0.1.6](https://panxii.duckdns.org/downloads/panxii-0.1.6-production.apk),
  56.137.117 bytes, SHA-256
  `C83E891A42AC20B5B199FBCFBF9273DDDE0CE2F1219BAFE1B4E08EC6D5D72F9C`.
  Download độc lập qua HTTPS khớp APK local. Signer SHA-256 vẫn
  `3D604285F3F0F18AB02EC1C1A3EA71832088F85FED4D13DA5E40C03A540F9EA7`;
  package `.production`, tên Panxii, versionName `0.1.6`, versionCode `8`.
  Không ghi đè APK 0.1.5 hoặc xóa dữ liệu thiết bị.
- Bản mới có phần đầu in A4/A5 gọn hơn, khách hàng chữ thường, QR tra cứu
  không nhãn, chân trang x/tổng trang; preview để trống thời gian Ngày in,
  khi gửi mới gắn timestamp, retry giữ timestamp/payload. Android tải OTA
  thành công sẽ restart tiến trình thật; tải lỗi không restart.
- 48 test update/dialog/in/designer pass, toàn bộ Flutter analyze sạch.
  Build từ working tree hiện tại gồm các thay đổi chưa commit; không gán
  nhầm source HEAD là bản đã chứa toàn bộ thay đổi.
- Đã cài và mở signed production trên Samsung SM-A336E. Native restart
  đã được kiểm tra riêng trên máy này bằng smoke debug tạm: PID 30008
  chuyển thành 30316, app mở lại; sau đó khôi phục app debug thường.
  Tại thời điểm phát hành native chưa xác nhận apply OTA; test patch 1
  phía trên đã xác nhận sau đó. Chưa nghiệm thu in giấy thực tế.
- Public policy latest `0.1.6`, URL APK mới; giữ minimum `0.1.0`.
  Chỉ recreate API cùng image cũ, không build/deploy backend source,
  migration, seed hay sửa dữ liệu nghiệp vụ. Worker, MySQL, RabbitMQ,
  Nginx và admin container IDs/images/mounts giữ nguyên. Public health
  HTTP 200 và private dependency readiness Healthy.
- Evidence chi tiết được lưu trong hồ sơ vận hành nội bộ.
  Rollback quảng bá bằng khôi phục latest/URL 0.1.5 và recreate riêng API;
  không hạ APK trên máy đã cài 0.1.6 hoặc rollback dữ liệu. Nếu lỗi app mới,
  phát hành native build cao hơn. Gate GO tổng thể vẫn giữ trạng thái cũ.

### Cập nhật thao tác bảo trì dữ liệu production

- Owner phê duyệt một lần bảo trì theo phạm vi đã xác nhận. Chi tiết nhận
  dạng và số liệu của tenant không được công bố.
- Backup được xác minh và giao dịch được diễn tập trước khi thực hiện. Audit
  được giữ lại; kết quả được kiểm tra sau bảo trì. Evidence chi tiết chỉ lưu
  trong hồ sơ vận hành nội bộ.
- Chính sách giá được áp dụng theo phạm vi phê duyệt; dữ liệu ngoài phạm vi
  được giữ nguyên. Thiết bị có thể còn cache cũ; đồng bộ và xử lý cache theo
  quy trình có phạm vi. Không chạy lại thao tác bảo trì.
### Triển khai cập nhật backend catalog pricing

- Bản backend đã vượt qua các kiểm tra build, tương thích và resilience trước
  triển khai. Các bộ kiểm thử tự động và local liên quan đều đạt.
- Backup mã hóa được xác minh trước migration. Thay đổi schema được áp dụng
  không seed hoặc reset dữ liệu; dữ liệu hiện có được kiểm tra để phát hiện
  thay đổi ngoài phạm vi.
- API và Worker đã được cập nhật; health check và monitoring đạt. Pricing
  policy chưa được bật trong đợt triển khai này.
- Hóa đơn offline có thể bị từ chối nếu giá catalog thay đổi trước khi hóa
  đơn được đồng bộ. Trước khi thay đổi giá, cần xác nhận thiết bị đã đồng bộ
  và xử lý draft đang chờ; queue rỗng trên server là chưa đủ.
- Commit, CI, backup, migration và bằng chứng triển khai chi tiết được lưu
  trong hồ sơ release nội bộ. Không rollback dữ liệu hoặc chạy migration
  downgrade như một hình thức rollback code.
### Mobile OTA patch 2: xem trước và in hóa đơn

- Đã phát hành stable patch `2` (`778305`) cho app
  `29ffa00b-3d56-412d-867f-146daf2528ad`, release `951572` / `0.1.5+7`;
  source mobile `cee827308f0bf86cf5c1643e91792d0513f85b14` trên nhánh
  `fix/invoice-preview-layout`. Truy vấn độc lập sau upload xác nhận
  channel `stable`, `is_rolled_back: false` lúc 23:39 ngày 04/11/2026.
- Android arm64, artifact `1763610`, 583299 bytes, SHA-256
  `25dea31e548bcb707acfd3ddc48c2ba4f1a3dcb4c2dda50014c4f80cc548288a`.
  Build bằng Flutter Shorebird `3.47.8`, giữ API HTTPS production;
  kiểm tra native/assets pass, không dùng override. APK native, signer,
  version policy và dữ liệu thiết bị không thay đổi.
- Thông tin khách hàng dùng cột trái cùng QR tra cứu, không đợi hết chiều cao
  QR. Header dịch vụ giữ nội dung tùy chỉnh, tiêu đề 16 pt đậm và mô tả tự wrap;
  đo dòng bằng cùng style khi render. PNG gửi in có ngày in ngay dưới ngày tạo,
  preview không giả định thời điểm in.
- 26 test preview/in và receipt design pass; `flutter analyze` sạch.
  Hot reload đã thử nhưng không có app kết nối. Chưa xác minh tải/apply patch
  trên điện thoại hoặc in giấy thực tế; phát hành active không đồng nghĩa
  thiết bị đã cập nhật. App dùng `auto_update: false`: chấp nhận cập nhật rồi
  khởi động lại khi không đang bán hàng.
- Patch tích lũy gồm client catalog từ `a00ebcf`; không deploy backend hoặc
  bật cờ làm tròn danh mục trong lần phát hành này. Chỉ dùng policy khi backend
  tương thích đã triển khai và trả cờ tương ứng. Không sửa snapshot hóa đơn.
- Nếu lỗi, rollback đúng patch `778305` trong Shorebird và xác minh trên
  thiết bị sau restart; rollback code không hoàn tác dữ liệu đã ghi.

### Backend EF10 đã deploy và dọn dữ liệu test theo owner

- Owner yêu cầu deploy bản đã merge và chuẩn bị database trống để nhập dữ
  liệu thật; xác nhận chỉ giữ SuperAdmin, xóa mọi tenant/TenantAdmin/Staff
  và dữ liệu test. Việc này là thao tác một lần được duyệt, không phải
  chức năng reset có thể chạy lại sau khi có dữ liệu thật.
- [PR #1](https://github.com/PanxiiPos/panxii-backend/pull/1) đã merge vào
  `feature/production-vps`; commit live
  `96e6ea73d17df1c0bf0a3e53325fe77c01dcd40d`.
  [CI đúng merge SHA 41842870171](https://github.com/PanxiiPos/panxii-backend/actions/runs/41842870171)
  success. API/Worker đã build trên VPS và chạy đúng image mới;
  kiểm tra deps thực tế EF Core `10.0.14`, MySql.EntityFrameworkCore `10.0.12`,
  không Pomelo. Runtime .NET `10.0.14`.
- Có manual backup verified trước deploy và khi quiesce trước xóa. Thử
  DELETE trong transaction rồi ROLLBACK trên dữ liệu thật pass trước COMMIT.
  Giữ FK enforcement, không TRUNCATE/drop/recreate schema/volume, không
  apply migration/seed. Giữ nguyên image cũ để rollback code; rollback code
  không khôi phục dữ liệu đã xóa.
- Final verify **21:25:25 UTC+7**: 1 dòng `platform_admins`, nguyên vẹn hash
  toàn dòng (gồm mật khẩu), 17 dòng migration history nguyên hash; cả
  **16 bảng còn lại = 0 dòng**, gồm tenant/user/device/admin session,
  catalog, invoice, outbox, sync, audit và settings. Không tạo invoice thử
  sau reset. Queue command/retry/DLQ đều rỗng; Worker có subscription.
- Public health/private readiness, monitor hiện có pass; unauthenticated
  platform tenant API trả 401. Dependency container IDs/mounts/volumes
  giữ nguyên; chỉ API/Worker thay image. API version policy latest `0.1.5`,
  minimum giữ `0.1.0`, Android URL trỏ APK 0.1.5 đã xác minh.
- Pilot soak đã disable/stop và có systemd condition yêu cầu marker
  `ALLOW_SYNTHETIC_PILOT` (không tồn tại), tránh vô tình nạp test lại.
  Không bật lại pilot hay dùng credential/session tenant test cũ. Thiết bị
  còn dữ liệu offline test chưa được xóa từ xa; phải dùng tài khoản tenant
  mới cho dữ liệu thật, không replay/nhập dữ liệu test cũ.
- Evidence root-only: `production-ef10-96e6ea7-deployed.json`,
  `production-ef10-96e6ea7-reset-before.json`,
  `production-ef10-96e6ea7-reset.json`, `production-ef10-96e6ea7-final.json`.
  Owner hoãn independent restore/external monitor; lịch backup và monitor
  đang có vẫn giữ nguyên, không coi các gate hoãn đã pass hay toàn bộ GO.

### Mobile production nội bộ 0.1.5 đã phát hành

#### Sửa tên hiển thị thành Panxii

- Theo owner, bỏ hậu tố Production khỏi launcher. Source `dbf3299`,
  Shorebird native release `951572` active, hiển thị `0.1.5`, Android
  versionCode `7`; không đổi `pubspec.yaml`, package hay signing certificate.
- [Link APK hiện tại](https://panxii.duckdns.org/downloads/panxii-0.1.5-production.apk)
  đã được thay atomic, tải lại qua HTTPS và xác minh signature/hash/manifest:
  label chính xác `Panxii`, package `.production`, versionName `0.1.5`,
  versionCode `7`, SHA-256
  `56CF3BC02864FEDF65C0D1E61EACE94813B655D72B3DEF9DE6F9FD9F775DE5FE`.
- Cài đè APK mới, không gỡ app hoặc clear data. Đổi tên native không áp dụng
  bằng OTA Dart patch; bản đã cài build 6 cần tải/cài lại bằng link trên.
  Version policy backend vẫn latest 0.1.5; so sánh semver không tự báo bản
  build 7 cho thiết bị đã chạy 0.1.5. Chưa xác minh cài trên thiết bị thật.
- APK build 6 được giữ dưới tên `panxii-0.1.5-build6-production.apk`;
  bằng chứng phía dưới là lịch sử bản build 6, không phải hash/link hiện tại.

#### Bằng chứng ban đầu của build 6

- Source mobile `399496a`, Shorebird release `951495` active, engine Flutter
  `3.47.8`. Phiên bản hiển thị `0.1.5`; Android versionCode `6` truyền riêng
  khi build, `pubspec.yaml` giữ `version: 0.1.5` theo yêu cầu owner.
- [APK production 0.1.5](https://panxii.duckdns.org/downloads/panxii-0.1.5-production.apk)
  đã upload atomic và tải lại qua HTTPS, SHA-256
  `631C59B127F735E0FEDFE5C6EF0EB8E66B4D25E86E8A9F987C4505663C2BD43B`.
  Chứng chỉ production giữ nguyên
  `3D604285F3F0F18AB02EC1C1A3EA71832088F85FED4D13DA5E40C03A540F9EA7`;
  package `com.panxiipos.panxii_mobile.production`, label Panxii Production.
  Cài cập nhật đè bản production 0.1.4, không clear data hoặc thay app debug.
- Bao gồm về danh sách hóa đơn sau lưu online/offline, catalog tự tải tiếp
  và footer không bị FAB che. 57 test liên quan pass, analyze sạch; một test
  quantity/services màn hẹp có lỗi từ baseline được loại riêng, không coi pass.
- Không có thiết bị ADB/app debug kết nối; hot reload đã thử nhưng không chạy
  được, chưa xác minh cài/UAT trên điện thoại cho bản này. Native release mới
  không phải OTA patch cho 0.1.4, không suy ra patch adoption/rollback của 0.1.5.
- Tại lúc phát hành APK chưa restart backend hoặc đổi policy; đợt deploy
  EF10 sau đó đã đặt latest `0.1.5`, minimum giữ `0.1.0` và URL APK mới,
  như bằng chứng backend bên trên.

### Bằng chứng nền và lịch sử trước 0.1.5

- Backend live application `848d8be0127890746269957873314f54b6971cbe`,
  deploy configuration nền `02623e1`, pilot runner/HTTPS keepalive overlay
  `b7ea06e`; feature branch chưa merge.
  .NET 10.0.14 đã deploy bằng quy trình backup-first, không migration/seed/reset
  volume; EF 8/Pomelo 8 tạm thời còn blocker lifecycle.
- `848d8be` Release build warning-free, 198 regression pass/4 opt-in skip;
  Docker resilience 23/23, cold local 1023,49/661,77/539,26 ms.
  [CI 41645651110](https://github.com/PanxiiPos/panxii-backend/actions/runs/41645651110)
  xanh build-test/resilience cho đúng SHA. Khởi tạo EF tracking trong bộ nhớ
  rồi clear, không SaveChanges; generated invoice API JSON giữ exact bytes/hash.
  Backup verified `manual/20261103T134454Z-c1942c4e6c30`, deploy chỉ API/Worker,
  giữ dependency IDs/mounts/volumes và rollback images. Required merge checks
  chưa xác nhận; chưa có auto-deploy production.
- Native Android `0.1.4+5`, Shorebird Flutter `3.47.8`, arm64, ký private
  production key; thiếu signing config sẽ fail release, không fallback debug key.
  Package `com.panxiipos.panxii_mobile.production`, label **Panxii Production**;
  cài song song trên Samsung SM-A336E, không di chuyển/xóa dữ liệu app debug.
  [Tải APK HTTPS](https://panxii.duckdns.org/downloads/panxii-0.1.4-production.apk).
  SHA-256 `A7BC51DCD3009994E8D502C2A3AE0A86C30FF31F977DEAAA88CA6AD61FEAA65A`;
  certificate SHA-256 `3D604285F3F0F18AB02EC1C1A3EA71832088F85FED4D13DA5E40C03A540F9EA7`.
  Khóa và file recovery mã hóa ngoài repo; owner xác nhận giữ mật khẩu.
  Production policy minimum `0.1.0`, latest `0.1.4`, có Android download URL thật.
- Shorebird patch 1 (`770453`) đã tải, verify signature, apply, full process
  restart và xác minh `current_patch_number=1`. Remote rollback + restart thực
  tế về base (`None`) pass, không clear app data. Ngày 04/11/2026 owner xác nhận
  thiết bị đã cập nhật lại thành công và Shorebird hiển thị 100%; chấp nhận
  gate reapply theo xác nhận owner, không phải một lần đo thiết bị mới.
  `auto_update: false`, người dùng xác nhận tải rồi khởi động lại.
- Owner xác nhận SuperAdmin login Swagger được. Chưa có UI SuperAdmin mobile;
  nghiệm thu quản trị qua API/Swagger, không yêu cầu một màn hình chưa triển khai.
  TenantAdmin physical-device offline/sync/restart/SignalR pilot còn pending.
- Soak bắt đầu 20:18:57 ngày 02/11 UTC+7 nhưng fail lúc 20:34:06 vì observed
  sync p95 1923,80 ms > 1500 ms; không có kết quả 24 giờ. Audit 102 invoice/
  receipt/change/published outbox khớp. Diagnostic chỉ replay cho thấy overhead
  kết nối mới. Ngày 03/11 backup
  verified rồi recreate API/Worker để áp dụng refresh-token policy 300 ngày
  theo `.env` owner đã sửa, cùng image/volumes. Owner duyệt reset username/
  password pilot bằng maintenance transaction có xác thực SuperAdmin/audit;
  login credentials mới pass, các phiên cũ thu hồi. Runner `d24ed97` dùng
  verified HTTPS keep-alive và authorized-user UUID guard. Lượt mới fail ngay
  first invoice 3038,62 ms. Owner duyệt startup reads, không warm-up invoice:
  `d2dc524` CI/local pass và đã deploy chỉ API/Worker, nhưng cold VPS 1755,38 ms
  vẫn vượt 1500 ms. Follow-up `6099f7a` chưa deploy vì cloud resilience cold
  gate fail (2052,85/1833,21/1771,00 ms), dù local 23/23 pass và regression
  198 pass/4 opt-in skip. Đây là lịch sử trước bản mới `848d8be`.
  Bản mới đã deploy sau CI pass; soak `20261103-135041` bắt đầu 20:50:49
  ngày 03/11 UTC+7, dùng ngay first invoice cold làm sample: **827,05 ms**.
  Không warm-up/discard sample; preserve failed state/mẫu cũ.
  Audit 13:54:54Z: 105 invoice/receipt/change/published outbox khớp, không
  duplicate/sai line values, command/retry/DLQ rỗng. Lượt này sau đó fail
  lúc 21:31:37 ngày 03/11 bởi `RemoteDisconnected` khi GET catalog,
  không phải vi phạm latency; chưa có 24h pass.
  Probe thật 70 giây idle tái hiện socket hết hạn (Nginx keepalive 65 giây).
  Runner `1ce4b50` reopen sau 50 giây, GET-only retry tối đa một lần có log,
  không retry ambiguous POST và vẫn tính đầy đủ reconnect latency;
  22 test pass Windows/VPS. Chỉ cập nhật runner, không recreate app.
  Run rút ngắn `20261104-042043` bắt đầu 11:20:49 ngày 04/11 UTC+7,
  dừng sinh tải lúc 19:00 theo owner (cleanup sau đó); first invoice
  177,53 ms trên process đang nóng, không coi là cold sample.
  Dự kiến 32 invoice mới theo nhịp cũ, **không phải 24h acceptance**.
  Audit trước run có 107 invoice/receipt/change/published outbox khớp,
  zero duplicate/sai line values, command/retry/DLQ rỗng.
  Run này fail **12:12:33 ngày 04/11**: read p95 **1344,62 ms > 500 ms**;
  4 invoice mới completed, sync max **707,71 ms**. Nginx catalog 25-40 ms;
  probe sau incident cho thấy DNS lookup trực tiếp thường 260-391 ms,
  không có phase timing để quy toàn bộ request lỗi cho DNS.
  Owner duyệt cache TTL `systemd-resolved` trên host (không pin IP/bỏ TLS):
  cached DNS **0,83-1,09 ms**, fresh TLS searches **40,54-323,01 ms**,
  probe sau 70 giây idle/TTL **353,11 ms**. Runner `ab82b1d` bổ sung lưu
  cửa sổ read/pull fail trước raise, không nới ngưỡng; **23 test Windows/
  37 test Linux pass**.
  Audit trước restart **111 invoice/detail/receipt/change/published outbox**
  khớp, zero duplicate/sai line values, queues rỗng. Run mới
  **`20261104-052131`**, **12:21:36 đến 19:00 UTC+7**, dự kiến 28 invoice
  mới theo nhịp cũ, first **189,63 ms** trên process nóng. Cửa sổ đầu
  read/pull p95 **64,44/13,12 ms**, host monitor pass. Không restart hay đổi
  app containers; failed run đã archive, không cộng dồn thời gian nghiệm thu.
  Run `20261104-052131` sau đó fail **16:58:32** vì read p95 **1316,50 ms**;
  20 invoice mới completed, sync max **530,60 ms**. Nginx 25-32 ms, cache
  DNS vẫn hoạt động nhưng idle cutoff 50 giây buộc reopen mỗi phút, trùng
  TTL 60 giây. Chưa có phase timing để quy spike hoàn toàn cho DNS.
  Owner duyệt public HTTPS keepalive **180 giây** / runner idle **120 giây**.
  `b7ea06e` thêm slow-request connect/remaining timing, không nới ngưỡng/
  đổi nhịp tải; **26 Windows / 40 Linux tests pass**. Nginx `-t`/graceful
  reload pass; probe 70 giây reuse đúng socket, search+pull **83,56 ms**;
  forced reconnect **330,70 ms**. Audit **131** invoice/detail/receipt/change/
  published outbox khớp, zero duplicate/sai line values, queues rỗng.
  Run mới **`20261104-100534`**, **17:05:38 đến 19:00 UTC+7**, dự kiến
  **8 invoice mới**, first **174,66 ms** nóng. Application/container IDs/
  images/mounts/start times không đổi, monitor pass. Failed run giữ nguyên
  archive. Kết quả kiểm tra 19:03:29: **bounded run hoàn thành 19:00:04**,
  service exit 0/success, **8/8 invoice mới**, cohort108. **109 cửa sổ**,
  max-window read/pull p95 **94,05/46,52 ms**, sync p95/max **300,57 ms**;
  zero reconnect/retry/transport fail. Final audit **139** invoice/detail/
  receipt/change/published outbox toàn tenant khớp, cohort108 đủ invoice/
  detail/completed receipt, zero duplicate/sai totals; queues rỗng, monitor
  pass. Bằng chứng root-only `pilot-keepalive-bounded-20261104-final-audit.json`.
  **Bounded workload + audit pass**, không phải 24h pass hoặc GO.
  Lúc **19:04 ngày 04/11 UTC+7**, owner chấp nhận kết quả bounded + audit,
  **miễn yêu cầu đủ 24 giờ và đóng gate synthetic soak/correctness** cho đợt
  này. Không đổi ngưỡng, không xóa failed evidence hoặc tự chạy lại cho đủ
  giờ; độ ổn định 24 giờ chưa được chứng minh. Hai read latency failures
  trước sửa vẫn là vi phạm end-to-end dù API/invoice xử lý nhanh.
  Physical-device pilot/UAT và các gate khác còn nguyên; chưa GO tổng thể.
  Chi tiết và gate còn lại tại
  [readiness](production_readiness_plan.md); chưa GO.

## Lịch sử release debug nội bộ (không phải bản production hiện tại)

Hướng triển khai nội bộ tiết kiệm chi phí và setup OTA để thực hiện sau:
[VPS thuê, APK nội bộ và Shorebird Free](../infrastructure/internal_free_deployment.md).
Shorebird Android `0.1.3+4`, release ID `934536`, đã publish và cài trên
Samsung SM-A336E. Settings hiển thị version/build hiện tại, ví dụ
`0.1.3+4`. APK nội bộ ký bằng debug key, chỉ dùng thử; không phân phối
production. Release `0.1.2+3`, `0.1.1+2` và `0.1.0+1` vẫn active trên
Shorebird. Chưa kiểm thử OTA patch hoặc triển khai pipeline/cloud production.

Kiểm tra ngày 2026-10-29: thiết bị vẫn cài `0.1.0+1`; API local trả
`minimumNativeVersion=latestNativeVersion=0.1.0` và `downloadUrls` rỗng, nên
không có native update để thông báo hoặc URL HTTPS để cài bản mới. Volume
MySQL local được tạo ngày 2026-10-28 và log ghi nhận database khởi tạo mới;
trước khi migrate, `PanxiiDev` không có bảng nào. Đã áp dụng 17 EF migrations
và nạp `seed-development.sql`; đây là dữ liệu demo mới, không phải bản khôi
phục dữ liệu cũ. Hiện không thấy volume MySQL cũ trong Docker hoặc file backup
trong workspace; log Docker không đủ để xác định thao tác nào đã thay volume.
Sau khi nạp seed, app đăng nhập và tải lại Home thành công. Policy đã khôi phục
về `0.1.0`; vì API không có URL tải HTTPS, các bản cũ không tự được thông báo
hoặc tải `0.1.2`. Đã cài APK mới trực tiếp qua ADB, giữ dữ liệu app; dialog chỉ
được kiểm tra với policy tạm thời. Source fix để `config/version` không phụ
thuộc session đã được đưa vào release mới, nhưng chưa kiểm thử lại với session
hết hạn.

Release `0.1.2+3` (ID `934520`) và `0.1.3+4` (ID `934536`) được build bằng
Shorebird/Flutter `3.47.8`. Bản `0.1.3+4` có APK
`mobile/build/app/outputs/flutter-apk/app-release.apk`,
SHA-256 `022C6CD5D8A4CF4C625721977ADF4C52ADDD1C7F46576554DEA4012EC4E18009`.
APK mới ký cùng chứng chỉ với bản cũ; cài đè bằng `adb install -r` và giữ dữ
liệu ứng dụng. Thiết bị báo versionCode `4`; Settings hiển thị `0.1.3+4`,
Shorebird khởi động, Home hiển thị tenant/dữ liệu mẫu và API trực tuyến.
Widget test Settings xác nhận version/build được render. Không phát hành OTA
patch vì Shorebird phát hiện assets bản dịch đổi so với release nền; APK đầy
đủ đã bao gồm assets. Release dùng debug key, chỉ cho thiết bị nội bộ; không
có URL APK HTTPS và không bật update policy lên `0.1.3`.

## 1. Versioning

- API major breaking change dùng `/api/v2`.
- Backend/mobile dùng semantic versioning.
- `minimumNativeVersion` chặn app native quá cũ.
- OTA Shorebird chỉ dùng cho thay đổi tương thích native code.

## 2. Release tracks

| Track | Mục đích |
|---|---|
| Development | Local Docker, seed giả |
| Staging | E2E, load, migration, restore drill |
| Pilot | Một nhóm tenant/thiết bị giới hạn |
| Production | Rollout có monitoring và rollback plan |

## 3. Release checklist

- Business rule và ADR đã chốt.
- OpenAPI, error catalog và i18n keys đồng bộ.
- Database migration đã review/backup.
- Unit, integration, contract, offline E2E pass.
- Security scan và dependency scan pass.
- Docker image pin theo git SHA.
- Health check và smoke test pass.
- Release notes có breaking/behavior changes.
- Rollback code và data plan được xác nhận.

## 4. Mobile compatibility

Server phải hỗ trợ mobile cũ trong khoảng thời gian đã chốt. Thêm field là backward-compatible; đổi/xóa field hoặc enum cần version/compatibility handling.

## 5. Rollout

1. Deploy staging.
2. Chạy migration expand.
3. Chạy smoke/contract/load test.
4. Pilot rollout.
5. Theo dõi 5xx, sync failure, duplicate, DLQ, latency và crash rate.
6. Mở rộng production.
7. Chỉ contract breaking mới tạo major version.

## 6. Rollback

- Code rollback bằng image tag trước.
- Data rollback không mặc định là migration down; dùng forward fix hoặc restore theo DR.
- Mobile rollback phải giữ API compatibility hoặc force update về bản hỗ trợ.
- OTA patch phải có kế hoạch disable/release patch mới nếu lỗi.

## 7. Release notes bắt buộc

Ghi rõ API changes, migration, feature flags, minimum app version, known issues, monitoring metrics và hướng xử lý khi sync lỗi.
