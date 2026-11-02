# RELEASE MANAGEMENT

Kế hoạch triển khai trước go-live:
[6 công việc production readiness](production_readiness_plan.md).
Đây là kế hoạch và điều kiện nghiệm thu, không phải xác nhận đã triển khai.

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
