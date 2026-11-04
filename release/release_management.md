# RELEASE MANAGEMENT

Kế hoạch triển khai trước go-live:
[6 công việc production readiness](production_readiness_plan.md).
Đây là kế hoạch và điều kiện nghiệm thu, không phải xác nhận đã triển khai.

## Bằng chứng release production cập nhật ngày 2026-11-04 — chưa GO

- Backend live application `848d8be0127890746269957873314f54b6971cbe`,
  deploy configuration `02623e1`, pilot runner `1ce4b50`; feature branch chưa merge.
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
  tế về base (`None`) pass, không clear app data. Patch đã roll forward server-side;
  chưa xác minh reapply trên thiết bị nên không coi roll-forward E2E đạt.
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
