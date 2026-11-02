# Hướng dẫn triển khai nội bộ tiết kiệm chi phí

- Cập nhật: 2026-11-02.
- Mục đích: lưu hướng dẫn và trạng thái triển khai từng thành phần.
  Chỉ các bước có xác nhận bên dưới đã được thực hiện; không coi toàn bộ
  hạ tầng là đã triển khai.
- Phạm vi: backend trên VPS thuê, APK Android nội bộ, Shorebird
  Free và Panxii Printer Bridge trên PC cửa hàng.
- Ngoài khoản thuê VPS đã được chủ dự án duyệt, không tự nâng gói, bật thanh
  toán vượt mức hoặc đăng ký thêm dịch vụ trả phí.

## 1. Trạng thái và lựa chọn đã chốt

| Hạng mục | Trạng thái khi viết hướng dẫn |
|---|---|
| Backend | API và Worker chạy sau reverse proxy; database và message broker chỉ truy cập qua mạng nội bộ. Chi tiết source/deploy identifier và cấu hình host được lưu trong hồ sơ vận hành riêng. |
| Máy chủ | Production chạy trên VPS Linux x64. Truy cập quản trị và firewall được giới hạn; không công bố nhà cung cấp, địa chỉ máy chủ hoặc cấu hình truy cập cụ thể. |
| Database VPS | Có dữ liệu pilot/test. Chi tiết tenant, số liệu giao dịch và lịch sử bảo trì được lưu trong evidence nội bộ, không công bố trong tài liệu public. |
| Secrets VPS | Secrets production được lưu ngoài repository và giới hạn quyền truy cập. Không công bố vị trí hoặc cấu hình lưu trữ chi tiết. |
| Backup | Backup được mã hóa và có bước xác minh; quy trình, lịch và thông tin khôi phục chỉ lưu trong hồ sơ vận hành riêng. |
| Domain/HTTPS | API production được phục vụ qua HTTPS; endpoint quản trị và dependency không public. Địa chỉ host và chi tiết mạng nội bộ không công bố tại đây. |
| Monitoring | Monitor trên host mỗi 5 phút kiểm tra TLS/health/services/disk/backup age; OnFailure backup/TLS/monitor gửi Discord. Inject lỗi và owner xác nhận nhận cảnh báo. Chưa có availability monitor độc lập ngoài VPS hoặc toàn bộ metrics queue/outbox/latency |
| CI/CD | Chưa tạo pipeline deploy production; mục tiêu là tự deploy backend khi merge `main` sau khi các bước kiểm tra đạt |
| Resilience gate | Ngày 2026-11-01 runner `backend/scripts/test-resilience.ps1` đã pass 22/22 test local Docker, không skip; có TRX/source manifest. Đã sửa publisher RabbitMQ phục hồi sau broker outage. Không chứng minh backup ngoài VM, HTTPS, SQLCipher trên thiết bị hoặc Shorebird OTA production; xem [ma trận kiểm thử](../testing/risk_based_testing_spec.md#10-gate-resilience-trước-triển-khai) |
| Development mobile gate | Ngày 2026-11-01 run `10e711ecd6964c088f22f96e0e7273c4` pass đủ 3+8 checkpoint trên `emulator-5554`; 8 invoice/local outbox Synced và MySQL không ghi trùng. Backup dữ liệu development hiện có restore khớp cả 18 bảng; source 488 file không đổi. API/Worker đã rebuild, các service phục hồi và app debug thường mở lại; không xóa volume. Chưa chứng minh backup ngoài VM, TLS hoặc Shorebird; xem [gate mobile](../testing/risk_based_testing_spec.md#11-gate-mobile-trên-development-đang-chạy) |
| Android | APK nội bộ `0.1.3+4` đã cài trên Samsung SM-A336E, versionCode `4`; Settings hiển thị đúng version/build; ký debug key chỉ dùng thử, không phân phối production |
| Shorebird | App `Panxii Internal`, `auto_update: false`; `0.1.3+4` active (release ID `934536`) và đã cài; `0.1.2+3` (ID `934520`), `0.1.1+2` (ID `934441`) và `0.1.0+1` (ID `931436`) vẫn active. Không phát hành patch vì assets dịch đã đổi; APK đầy đủ bao gồm assets. Chưa thử tải APK qua HTTPS |
| Update policy | API local trả minimum/latest `0.1.0`, `downloadUrls` rỗng; app cũ không tự được thông báo/tải `0.1.3`. Dialog native từng xác nhận với policy tạm thời `0.1.1`; chưa bật policy mới do thiếu URL HTTPS |
| Shorebird CLI | Đã cài và đăng nhập trên máy phát triển hiện tại; máy khác cần cài/đăng nhập riêng |
| Printer Bridge | Có bản EXE Windows và source trên nhánh feature; chưa xác nhận in Canon thật và tự chạy sau đăng nhập tại cửa hàng |

Source đã push trên nhánh feature **không có nghĩa đã merge hoặc deploy**.
HTTPS và backup ngoài VM đã có smoke/restore evidence riêng như bảng trên;
không suy ra tenant pilot, hai lịch backup tự động hay toàn bộ public production đã đạt.
Quy trình lặp lại và cấu hình hiện có tại
[backend deploy](../../backend/deploy/README.md).
Các ví dụ dưới đây là thao tác cần làm sau; không dán nguyên ví dụ có placeholder
vào production. Không gửi mật khẩu, private key, keystore hoặc token vào chat/Git.

**Review trước GO:** .NET 8 EOL ngày 2026-11-10 theo
[Microsoft release metadata](https://raw.githubusercontent.com/dotnet/core/main/release-notes/releases-index.json).
Cần chốt nâng runtime/framework, review lifecycle MySQL/RabbitMQ và kiểm thử
tương thích; image đã pin không đồng nghĩa còn được hỗ trợ. Không tự nâng major
database trên volume hiện có hoặc coi cập nhật OS là đã xử lý dependency này.

## 2. Chi phí và giới hạn miễn phí

| Thành phần | Hướng dùng | Điểm cần kiểm tra trước khi đăng ký |
|---|---|---|
| Máy chủ | Production chạy trên VPS Linux x64. Truy cập quản trị và firewall được giới hạn; không công bố nhà cung cấp, địa chỉ máy chủ hoặc cấu hình truy cập cụ thể. |
| Backend | API và Worker chạy sau reverse proxy; database và message broker chỉ truy cập qua mạng nội bộ. Chi tiết source/deploy identifier và cấu hình host được lưu trong hồ sơ vận hành riêng. |
| HTTPS | Let's Encrypt | Cần tên miền hợp lệ, DNS và cơ chế gia hạn; domain không mặc định miễn phí |
| APK | Phát hành nội bộ, không cần Google Play ở giai đoạn đầu | Cần kênh tải HTTPS tin cậy và khóa ký APK ổn định |
| OTA | Shorebird Free | Hiện có 5.000 lượt **cài bản vá/tháng**, không phải 5.000 thiết bị |
| Tự động hóa | GitHub Actions trong hạn mức tài khoản hiện có | Repo private có giới hạn phút/lưu trữ; kiểm tra cả chính sách registry, băng thông và artifact |
| Backup | Backup được mã hóa và có bước xác minh; quy trình, lịch và thông tin khôi phục chỉ lưu trong hồ sơ vận hành riêng. |
| In | Printer Bridge trên PC hiện có | Không đưa bridge hay cổng in lên VPS/Internet |

Đây không phải cam kết tổng chi phí bằng 0: VPS là chi phí định kỳ đã được chấp
nhận. Kiểm tra hạn mức, điều khoản và giá tại thời điểm thực hiện. Theo dõi sử
dụng; cảnh báo ngân sách không phải cơ chế tự chặn mọi khoản phí. Không tự nâng
cấu hình/gói VPS hoặc mua thêm dịch vụ nếu chưa được chủ dự án duyệt.

## 3. Chuẩn bị VPS thuê

> Oracle Cloud Always Free đã bị loại ngày 2026-11-02 vì không đăng ký được
> tài khoản. Phương án hiện tại là thuê VPS trả phí.

1. Chọn nhà cung cấp và gói VPS phù hợp với ngân sách, yêu cầu bảo mật,
   backup và chính sách lưu trữ; chi tiết lựa chọn production được giữ nội bộ.
2. Đánh giá cấu hình tổng thể cho API/Worker/MySQL/RabbitMQ/Nginx trên môi
   trường phù hợp; đo tải trước khi quyết định điều chỉnh tài nguyên.
3. Tạo VM Ubuntu LTS còn được hỗ trợ, ví dụ Ubuntu 24.04, image đúng kiến trúc
   CPU của gói VPS (thường x64/amd64).
4. Đăng nhập SSH bằng key, tạo tài khoản quản trị/deploy phù hợp, cập nhật OS.
   Chặn đăng nhập SSH bằng mật khẩu/root sau khi đã thử tài khoản thay thế.
5. Cấu hình firewall của nhà cung cấp (nếu có) và firewall OS: public `80/443`;
   `22` chỉ từ nguồn quản trị được kiểm soát. Không mở `3306`, `5672`, `15672`,
   `8080`.
6. Cài Docker Engine và Compose plugin theo tài liệu Docker cho Ubuntu.
   Quyền truy cập Docker tương đương quyền quản trị máy; không cấp rộng rãi.
7. Cấu hình đồng bộ giờ, log rotation, giám sát đĩa/RAM và dung lượng Docker.
8. Lên lịch nhắc gia hạn thanh toán; VPS trễ hạn có thể bị khóa hoặc xóa dữ
   liệu, nên backup ngoài VM là bắt buộc trước khi nhận dữ liệu thật.

**Điều kiện tiếp tục:** VPS đã thuê trong ngân sách được duyệt, SSH hoạt động,
đã có nơi backup ngoài VM và đã kiểm tra phương án khi mất VM hoặc nhà cung cấp
gặp sự cố.

### 3.1. Backup MySQL lên Google Drive bằng Rclone

Chính sách đã chốt ngày 2026-11-02, **chưa triển khai**:

- Lịch: 06:00, 12:00 và 21:00 giờ Việt Nam (UTC+7). Nếu cron của VM chạy theo
  UTC, tương ứng `23:00` (ngày hôm trước), `05:00` và `14:00` UTC; xác nhận
  timezone thực tế của VM trước khi đặt lịch.
- Luồng mỗi lượt: dump MySQL nhất quán → nén → **mã hóa** (AES-GCM hoặc remote
  `crypt` của Rclone) → upload Google Drive qua Rclone → xác minh artifact tồn
  tại, đúng hash và tải về đọc lại được.
- Retention: chỉ giữ **6 bản mới nhất** trên Drive (khoảng 2 ngày lịch sử);
  bản cũ hơn tự xóa để tiết kiệm dung lượng. Chỉ xóa bản cũ **sau khi** bản mới
  đã upload và xác minh thành công; lượt backup lỗi không được xóa gì.
- Backup thủ công trước deploy/migration lưu ở thư mục riêng, không thuộc chu
  kỳ tự xóa này.
- Token/credential Rclone và key giải mã lưu ngoài Git, có bản sao phục hồi
  độc lập với VM; mất VM không được đồng nghĩa mất khả năng giải mã backup.
- Giới hạn đã chấp nhận: khoảng cách lớn nhất giữa hai lượt là 9 giờ
  (21:00 → 06:00 hôm sau và 12:00 → 21:00) nên có thể mất tới ~9 giờ dữ liệu
  server; giữ 6 bản nghĩa là lịch sử khoảng 2 ngày, hỏng ngầm phát hiện muộn
  hơn 2 ngày có thể không còn bản tốt. Chủ dự án chấp nhận giới hạn này.
- Tham chiếu yêu cầu nghiệm thu backup/restore tại
  [kế hoạch production](../release/production_readiness_plan.md) và
  [disaster recovery](disaster_recovery.md).

## 4. Backend, DNS và HTTPS

Đọc [hạ tầng chuẩn](infrastructure_setup.md) và
[khôi phục sự cố](disaster_recovery.md) trước khi triển khai.

1. Chọn tên miền/subdomain do mình kiểm soát, trỏ DNS tới IP VM và cấu hình
   Nginx + chứng chỉ HTTPS hợp lệ. Nếu chưa có domain, dừng ở bước chuẩn bị;
   không cho app production dùng HTTP hoặc bỏ xác minh chứng chỉ.
2. Tạo cấu hình Compose production riêng cho API, Worker, MySQL, RabbitMQ,
   Nginx và volume dữ liệu bền vững. **Không dùng nguyên
   `docker-compose.local.yml` hoặc các ví dụ local trong tài liệu cũ.**
3. Đặt môi trường API là `Production`; đối chiếu tên biến mà code đang đọc:
   `DB_CONNECTION`, `RABBITMQ_CONNECTION`, `JWT_SECRET_KEY`, `JWT_ISSUER`,
   `JWT_AUDIENCE`. Worker dùng đúng cấu hình DB/broker mà phiên bản đó yêu cầu.
   Không suy diễn rằng các tên `ConnectionStrings__...` mẫu đã được code hỗ trợ.
4. Tạo thông tin xác thực riêng, không tái dùng tài khoản local. Secret nằm ở
   secret store hoặc tệp ngoài Git có quyền hạn chế. Không tự thêm hậu tố
   `_FILE` nếu ứng dụng chưa có logic đọc secret file tương ứng.
5. API, MySQL và RabbitMQ chỉ trong Docker network riêng; chỉ Nginx public.
   Bảo đảm container vẫn có outbound cần thiết cho DNS/dịch vụ ngoài.
6. Build image API/Worker đúng kiến trúc CPU của VPS (thường `linux/amd64`);
   kiểm tra manifest của image MySQL/RabbitMQ/Nginx theo đúng tag được chọn.
   Nếu VPS là ARM64, phải build và kiểm tra lại toàn bộ image theo kiến trúc đó.
7. Pin image ứng dụng theo Git SHA, pin phiên bản image hạ tầng; không deploy
   `latest`. Registry phải có quyền pull phù hợp nếu image private.
8. Review migration, backup trước khi áp dụng và dùng tài khoản migration
   riêng. Không tự chạy migration phá hủy dữ liệu khi app khởi động.
9. Tạo tài khoản quản trị ban đầu theo cơ chế provisioning được kiểm tra;
   không bật seed dữ liệu demo hoặc mật khẩu mặc định trên production.
10. Kiểm tra `/health/live`, `/health/ready`, `/api/v1/health`, đăng nhập,
    tenant isolation, SignalR `/hubs/pos`, lập hóa đơn offline và đồng bộ lại.

## 5. GitHub Actions: tự deploy backend khi merge `main`

Đây là **thiết kế pipeline cần triển khai sau**, chưa có workflow production
hoàn chỉnh để bật ngay. Tuân theo [quy trình PR](../development_workflow.md):
test và xác nhận trước khi merge, không push thẳng `main`.

### Cấu hình trên GitHub

Trong repository backend, tạo Environment `production` nếu gói GitHub hỗ trợ;
nếu không, dùng repository secrets và kiểm soát quyền repository/branch.
Khả năng dùng protection rules ở repo private phụ thuộc gói, không mặc định
coi GitHub Free có đầy đủ approval gates.

Tên dưới đây là **quy ước dự kiến**, chỉ có tác dụng khi workflow được viết:

| Loại | Tên dự kiến | Nội dung |
|---|---|---|
| Secret | `DEPLOY_SSH_PRIVATE_KEY` | Khóa riêng của tài khoản deploy, không dùng khóa cá nhân chính |
| Secret | `DEPLOY_KNOWN_HOSTS` | Host key SSH đã đối chiếu qua kênh tin cậy; không tắt host verification |
| Variable | `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PORT` | Đích SSH, tên tài khoản và cổng |
| Variable | `DEPLOY_PATH` | Thư mục triển khai trên VM |
| Variable | `API_HEALTH_URL` | URL HTTPS để kiểm tra sức khỏe sau deploy |

Thông tin DB/JWT/broker nên giữ trên VM/secret store, không truyền lại trong
mọi lượt build. Với GHCR, dùng `GITHUB_TOKEN` quyền tối thiểu khi push nếu phù
hợp; VM cần credential pull riêng nếu image private. Kiểm tra quota và chi phí
registry trước khi chọn. Không đưa credential vào image, artifact hoặc log.

### Luồng thực thi cần có

1. PR: restore, build, unit/integration/contract tests và scan phụ thuộc.
2. Push vào `main` sau merge: lặp lại các gate bắt buộc; fail thì không deploy.
3. Build/push image API/Worker theo SHA đúng kiến trúc CPU của VPS; cấu hình
   timeout và retention
   ngắn cho artifact. Pin action bên thứ ba vào commit đã review.
4. Serialize lượt deploy production; không hủy tiến trình đang migration để
   nhường lượt mới.
5. Xác minh SSH host, lấy backup, kiểm tra migration đã review rồi mới áp dụng.
6. Pull image và cập nhật dịch vụ, giữ volume DB/broker.
7. Health/smoke test; nếu lỗi, rollback image trước. Không tự migration down;
   khôi phục dữ liệu theo kế hoạch đã kiểm thử.

Khởi tạo production lần đầu phải có kiểm tra thủ công/pilot theo
[release management](../release/release_management.md), không bỏ qua chỉ vì
các lần deploy sau được tự động hóa.

## 6. Android APK nội bộ và Shorebird Free

### 6.1. Chuẩn bị trước, chưa cần đăng nhập

- Giữ package `shorebird_code_push` và luồng kiểm tra/tải OTA hiện có.
- Không tạo `app_id` giả hoặc khẳng định OTA đã hoạt động khi chưa có release.
- Chưa cần chạy `login`, `init`, `release` hay `patch` khi chỉ đọc hướng dẫn này.
- Kiểm tra Flutter của dự án và phiên bản Shorebird hỗ trợ trước khi build.
  Tại thời điểm kiểm tra: Flutter dự án 3.47.7; CLI Shorebird 1.6.133 đi kèm
  Flutter 3.47.8. Không tự nâng SDK mà chưa kiểm thử.

### 6.2. Khi sẵn sàng: cài CLI, đăng nhập và khởi tạo app

Cài CLI theo [hướng dẫn chính thức](https://docs.shorebird.dev/getting-started/).
Trên máy Windows đã cài ở vị trí mặc định, chạy từ thư mục mobile:

```powershell
Set-Location D:\Panxii\mobile
$shorebird = Join-Path $env:USERPROFILE '.shorebird\bin\shorebird.ps1'
& $shorebird doctor
& $shorebird login
& $shorebird init --display-name Panxii
```

Đăng nhập bằng trình duyệt, không gửi token/mã xác thực vào chat. Nếu execution
policy chặn script, có thể gọi qua
`powershell.exe -NoProfile -ExecutionPolicy Bypass -File` cho tiến trình đó;
không cần đổi policy toàn hệ thống.

`init` tạo app thật, `shorebird.yaml` có `app_id` và thêm asset trong
`pubspec.yaml`. `app_id` không phải secret, được commit cùng cấu hình. Không
dùng `init --force` để “sửa lỗi” vì có thể tạo app mới.

App đã có UI kiểm tra/tải patch và thông báo khởi động lại; thêm
`auto_update: false` vào `shorebird.yaml` để tránh updater nền mặc định vượt qua
luồng người dùng xác nhận. Giữ nguyên `app_id` thật vừa sinh.

### 6.3. Ký APK release và phát hành bản nền

#### Bản thử nội bộ đã chốt ngày 2026-10-28

- Shorebird app: `Panxii Internal`, app_id
  `29ffa00b-3d56-412d-867f-146daf2528ad`.
- Android `0.1.0+1`, API `http://127.0.0.1:5099/api/v1`, chỉ dùng trên
  điện thoại thử nghiệm có `adb reverse tcp:5099 tcp:5099` tới PC chạy API.
  Đây không phải URL dùng độc lập qua Internet.
- Giữ debug key hiện tại theo lựa chọn của chủ dự án để cài đè bản thử mà
  không xóa dữ liệu. **Không phân phối bản này cho production.**
- `auto_update: false` để tải patch qua luồng xác nhận đã có trong app.
- Đã xác nhận release ID `931436`, Android `0.1.0+1` active, Flutter
  `3.47.8` (revision `baf23dc375`). Chủ dự án thực hiện phát hành qua CLI.
- Đã cài đè APK bằng `adb install -r` lên Samsung SM-A336E, đối chiếu
  chữ ký trước khi cài và SHA-256 APK đã cài khớp artifact:
  `EE6B094A881A37754FDA845054A6116D88E60777D0DD13FDAA3B79C291C3A46A`.
  App khởi động, updater Shorebird khởi tạo thành công, chưa có patch active;
  API local readiness HTTP 200. Không gỡ app hoặc xóa dữ liệu.
- Việc tạo app_id không phải phát hành: chỉ xác nhận OTA hoạt động sau khi
  có release nền, cài đúng APK và kiểm tra một patch thật trên thiết bị.
- Không gỡ bản cũ hoặc xóa dữ liệu nếu chữ ký không khớp. Khi chuyển sang
  production phải chuẩn bị release keystore, API HTTPS và kế hoạch dữ liệu.
- Không chạy `flutter run` lên thiết bị đang kiểm thử OTA vì sẽ thay bản
  Shorebird bằng bản debug thông thường. Sau khi quay lại phát triển debug,
  cần cài lại APK nền Shorebird để thử OTA.

#### Thử native update ngày 2026-10-29 — thông báo chưa xác nhận

- Shorebird release Android `0.1.1+2`, ID `934441`, active; Flutter `3.47.8`
  (revision `6e5e949520159b5ceb029598be37520cd433b046`). APK thử nội bộ được ký
  bằng debug key như lựa chọn hiện tại; SHA-256 artifact:
  `BB8FB500DE157F2C38D94482E4F65E1BD52FB742155F436B03AE2888210E87E6`.
- Không cài APK mới. Samsung vẫn ở `0.1.0` / versionCode `1`, giữ nguyên app
  và dữ liệu hiện có.
- Đã tạm đặt API local `latestNativeVersion=0.1.1` trong lúc thử rồi khôi phục
  về `0.1.0`; không đổi `minimumNativeVersion` hay cấu hình production.
  App không hiện dialog cập nhật; UI gặp trạng thái retry/lỗi tải cấu hình cửa
  hàng. Vì vậy không xác nhận client đã nhận policy phiên bản mới.
- App thử dùng API `http://127.0.0.1:5099/api/v1` qua `adb reverse`. APK có
  `targetSdk 36`; Android manifest hiện không khai báo
  `android:usesCleartextTraffic` hoặc `networkSecurityConfig`. HTTP cleartext
  bị Android chặn mặc định với target mới, nên đây là nguyên nhân khả dĩ cần
  kiểm tra; chưa có log xác nhận request cụ thể bị chặn. Không bật cleartext
  rộng cho production. Lần thử lại cần API HTTPS mà điện thoại tin cậy, hoặc
  một cấu hình kiểm thử chỉ giới hạn local đã được review.
- Sau thử nghiệm, API health vẫn `ok`, policy trả minimum/latest `0.1.0`,
  port forwarding tạm đã gỡ, và không có thay đổi mã nguồn mobile/backend.

Các bước ký release dưới đây áp dụng cho bản phân phối thật, không phải
ngoại lệ debug key của bản thử nội bộ trên.

1. Tạo Android release keystore riêng, sao lưu mã hóa ở nơi an toàn. Không commit
   `.jks`, `.keystore`, mật khẩu hay `android/key.properties`.
2. Thay cấu hình release đang dùng debug key trong
   `android\app\build.gradle.kts` bằng signing config release thật. Thiếu khóa
   thì build release phải báo lỗi, không âm thầm quay về debug key.
3. Chọn phiên bản native/build number, URL API HTTPS thật và Flutter được
   Shorebird hỗ trợ. Kiểm tra `HOME_PREVIEW` không bật trong bản phát hành.
4. Chạy kiểm thử rồi tạo APK bằng **Shorebird**, không chỉ `flutter build apk`.
   Ví dụ dưới đây dùng placeholder; thay bằng giá trị đã chốt trước khi chạy:

```powershell
& $shorebird release android --artifact apk --flutter-version '<supported-flutter-version>' '--' '--dart-define=API_BASE_URL=https://<api-domain>/api/v1'
```

Dùng artifact do lệnh báo ra, đối chiếu SHA-256 và chữ ký trước khi phân phối
qua HTTPS tin cậy. Cài thử trên thiết bị nội bộ. Nếu trước đó cài APK debug,
chữ ký mới có thể không cập nhật đè được; **không gỡ app khi còn hóa đơn/outbox
chưa đồng bộ**. Lập kế hoạch chuyển dữ liệu và kiểm tra đồng bộ trước.

### 6.4. Tạo bản vá OTA sau này

Chỉ vá Dart tương thích với native/API của release nền. Thay đổi Kotlin/Java,
Gradle, AndroidManifest, plugin native, Flutter engine hoặc assets (bao gồm
JSON bản dịch được đóng gói) phải đánh giá tạo APK release mới; không ép bỏ
qua kiểm tra native/assets của Shorebird.

Giữ đúng release version **gồm build number**, SDK, API URL và build arguments
của bản nền. Ví dụ:

```powershell
& $shorebird patch android --release-version '<version+build>' '--' '--dart-define=API_BASE_URL=https://<api-domain>/api/v1'
```

Không dùng `latest` khi có nhiều release vì dễ nhắm sai bản. Kiểm tra pilot
trước khi mở rộng; tải xong chỉ áp dụng sau lần khởi động app tiếp theo, không
tự khởi động lại khi đang bán hàng. Mất mạng phải giữ dữ liệu và cho thử lại.
Nếu patch lỗi, rollback trong Shorebird Console theo release/patch đã xác
định rồi kiểm tra trên thiết bị; rollback code không hoàn tác dữ liệu đã ghi.
Shorebird không cập nhật backend hoặc EXE Printer Bridge.

### 6.5. Đưa Shorebird vào GitHub Actions sau khi đã có release

- Dùng action chính thức `shorebirdtech/setup-shorebird`, pin phiên bản/commit
  đã review. Chọn đúng Flutter như bản nền.
- Tạo API key trong Shorebird Console, lưu vào GitHub Secret
  **`SHOREBIRD_TOKEN`**. Không dùng `shorebird login:ci`: token kiểu cũ đã ngừng
  hoạt động từ 2026-10-01 theo tài liệu hiện tại.
- Gói Free dùng API key; không thiết kế dựa vào OIDC trust rules của gói
  Business/Enterprise.
- Signing secrets dự kiến: `ANDROID_KEYSTORE_BASE64`,
  `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`.
  Base64 không phải mã hóa; giải mã vào tệp tạm, không upload artifact chứa khóa.
- Các giá trị không nhạy cảm như API URL/release version lưu trong Variables
  hoặc input workflow; phiên bản patch phải chỉ rõ bản nền.
- Backend tự deploy sau merge `main` là lựa chọn đã chốt; **không mặc định mỗi
  merge mobile là một patch**. Lúc đầu nên có workflow phát hành/patch chạy
  thủ công để phân biệt Dart-only với thay đổi cần APK và tránh dùng quota vô ích.
- Theo dõi lượt cài patch trên Console, retention artifact và phút Actions.
  Không bật nâng gói/vượt mức tính phí mà chưa được chấp thuận.

## 7. Printer Bridge và kiểm tra cuối

Lấy source/bản build đúng phiên bản từ
[repository Printer Bridge](https://github.com/PanxiiPos/panxii-printer-bridge/tree/feature/desktop-printer-bridge).
Đọc README của repository đó để setup EXE, driver Canon, ghép nối token/pin,
mở TCP 8765 + UDP 8766 **Private/LocalSubnet** và bật tự chạy sau đăng nhập.
Không mở các cổng này trên VPS/router Internet.

Trước khi sử dụng thật:

- [ ] Backup MySQL ngoài VM và thử restore vào môi trường tách biệt.
- [ ] Chạy gate resilience trên stack cô lập; kiểm tra TRX/manifest và bằng chứng Flutter/SQLCipher, backup ngoài VM, TLS, Shorebird riêng. Không chạy chaos trên VM production.
- [ ] Xác nhận toàn bộ tài nguyên cloud/CI/storage còn trong mức chi phí chấp nhận.
- [ ] HTTPS và gia hạn chứng chỉ hoạt động; không công khai DB/broker.
- [ ] Đăng nhập, offline → online, sync/retry và tenant isolation đạt.
- [ ] APK release ký đúng khóa, nhận patch Shorebird rồi áp dụng sau khởi động lại.
- [ ] Có bản APK đầy đủ để phục hồi khi OTA không xử lý được.
- [ ] In Canon thật, kiểm tra A4/QR và trạng thái lệnh in.
- [ ] Tắt/bật PC, đăng nhập đúng tài khoản: bridge tự chạy.
- [ ] Thử mạng chưa sẵn sàng lúc đăng nhập; lỗi phải rõ ràng, có thể thử lại.
- [ ] Đổi IP PC, app tìm lại đúng chứng chỉ/token, thử lại không in trùng.
- [ ] Đã thử rollback backend và biết cách rollback patch, không xóa dữ liệu.

## 8. Tài liệu chính thức để đối chiếu lúc thực hiện

Giới hạn và lệnh có thể thay đổi; kiểm tra lại trước khi dùng:

- [Docker Engine trên Ubuntu](https://docs.docker.com/engine/install/ubuntu/).
- [Shorebird pricing](https://shorebird.dev/pricing/).
- [Khởi tạo Shorebird](https://docs.shorebird.dev/code-push/initialize/).
- [Tạo release](https://docs.shorebird.dev/code-push/release/).
- [Tạo patch](https://docs.shorebird.dev/code-push/patch/).
- [Shorebird GitHub Actions](https://docs.shorebird.dev/code-push/ci/github/).
- [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions).