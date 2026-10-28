# Hướng dẫn triển khai nội bộ tiết kiệm chi phí

- Cập nhật: 2026-10-26.
- Mục đích: lưu các bước để thực hiện sau, **không phải xác nhận đã triển khai**.
- Phạm vi: backend trên Oracle Cloud Always Free, APK Android nội bộ, Shorebird
  Free và Panxii Printer Bridge trên PC cửa hàng.
- Không tự nâng gói, bật thanh toán vượt mức hoặc đăng ký dịch vụ trả phí.

## 1. Trạng thái và lựa chọn đã chốt

| Hạng mục | Trạng thái khi viết hướng dẫn |
|---|---|
| Backend | Đang chạy local; có Dockerfile API/Worker, chưa có cấu hình Compose production hoàn chỉnh |
| Oracle Cloud | Chọn thử Always Free; chưa xác nhận đã tạo VM, IP hoặc dịch vụ |
| Domain/HTTPS | Chưa cấu hình |
| CI/CD | Chưa tạo pipeline deploy production; mục tiêu là tự deploy backend khi merge `main` sau khi các bước kiểm tra đạt |
| Android | Phân phối APK nội bộ trước; cấu hình release hiện còn dùng debug key, phải thay trước khi phát hành |
| Shorebird | Có code kiểm tra/tải OTA trong app; chưa khởi tạo app, chưa có `shorebird.yaml`/`app_id`, chưa có release hoặc patch |
| Shorebird CLI | Đã cài trên máy phát triển hiện tại, chưa đăng nhập; máy khác cần cài riêng |
| Printer Bridge | Có bản EXE Windows và source trên nhánh feature; chưa xác nhận in Canon thật và tự chạy sau đăng nhập tại cửa hàng |

Source đã push trên nhánh feature **không có nghĩa đã merge hoặc deploy**.
Các ví dụ dưới đây là thao tác cần làm sau; không dán nguyên ví dụ có placeholder
vào production. Không gửi mật khẩu, private key, keystore hoặc token vào chat/Git.

## 2. Chi phí và giới hạn miễn phí

| Thành phần | Hướng dùng | Điểm cần kiểm tra trước khi đăng ký |
|---|---|---|
| Máy chủ | Oracle Always Free, ưu tiên A1 nếu còn tài nguyên | Hạn mức theo tài khoản/region; có thể hết máy hoặc thu hồi VM ít dùng |
| Backend | Docker Engine/Compose, MySQL, RabbitMQ, Nginx tự quản lý trên VM | Tốn tài nguyên VM; tự chịu trách nhiệm bảo mật, sao lưu và khôi phục |
| HTTPS | Let's Encrypt | Cần tên miền hợp lệ, DNS và cơ chế gia hạn; domain không mặc định miễn phí |
| APK | Phát hành nội bộ, không cần Google Play ở giai đoạn đầu | Cần kênh tải HTTPS tin cậy và khóa ký APK ổn định |
| OTA | Shorebird Free | Hiện có 5.000 lượt **cài bản vá/tháng**, không phải 5.000 thiết bị |
| Tự động hóa | GitHub Actions trong hạn mức tài khoản hiện có | Repo private có giới hạn phút/lưu trữ; kiểm tra cả chính sách registry, băng thông và artifact |
| Backup | Lưu bản sao mã hóa ngoài VM bằng dung lượng sẵn có hoặc gói miễn phí phù hợp | Không coi volume/snapshot cùng VM là bản backup độc lập |
| In | Printer Bridge trên PC hiện có | Không đưa bridge hay cổng in lên VPS/Internet |

Đây không phải cam kết tổng chi phí luôn bằng 0. Kiểm tra hạn mức, điều khoản và
giá tại thời điểm thực hiện. Theo dõi sử dụng; cảnh báo ngân sách không phải cơ
chế tự chặn mọi khoản phí. Không nâng Oracle lên Pay As You Go chỉ để vượt lỗi
thiếu tài nguyên nếu chưa chấp nhận rủi ro chi phí.

## 3. Chuẩn bị Oracle Always Free

1. Tạo tài khoản tại trang Oracle chính thức, kiểm tra yêu cầu xác minh và chọn
   **home region** phù hợp ngay từ đầu. Xác nhận các tài nguyên mang nhãn Always
   Free, không nhầm với credit dùng thử có thời hạn.
2. Kiểm tra hạn mức Compute/Block Volume hiện tại trong Console. Nếu A1 báo
   thiếu tài nguyên, thử lại sau hoặc region/availability domain được hỗ trợ;
   không tự đổi sang shape trả phí.
3. Tạo VM Ubuntu LTS còn được hỗ trợ, ví dụ Ubuntu 24.04. A1 là **ARM64**, không
   phải x64; chọn image OS đúng kiến trúc.
4. Chỉ cấp CPU/RAM/boot volume trong tổng hạn mức miễn phí của tài khoản.
   Không dùng số liệu từ bài hướng dẫn cũ làm cam kết cấp phát.
5. Đăng nhập SSH bằng key, tạo tài khoản quản trị/deploy phù hợp, cập nhật OS.
   Chặn đăng nhập SSH bằng mật khẩu/root sau khi đã thử tài khoản thay thế.
6. Cấu hình cả OCI Security List/NSG và firewall OS: public `80/443`; `22` chỉ
   từ nguồn quản trị được kiểm soát. Không mở `3306`, `5672`, `15672`, `8080`.
7. Cài Docker Engine và Compose plugin theo tài liệu Docker cho Ubuntu.
   Quyền truy cập Docker tương đương quyền quản trị máy; không cấp rộng rãi.
8. Cấu hình đồng bộ giờ, log rotation, giám sát đĩa/RAM và dung lượng Docker.
   Không tạo tải giả để tránh chính sách thu hồi VM nhàn rỗi.

**Điều kiện tiếp tục:** VM còn trong hạn mức Free, SSH hoạt động, đã có nơi
backup ngoài VM và đã kiểm tra phương án khi Oracle thu hồi/mất máy.

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
6. Build image API/Worker hỗ trợ `linux/arm64` cho A1; kiểm tra manifest của
   image MySQL/RabbitMQ/Nginx theo đúng tag được chọn. Không giả định image
   build trên Windows x64 sẽ chạy trên ARM64.
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
3. Build/push image API/Worker theo SHA với ARM64; cấu hình timeout và retention
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

- [Oracle Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm).
- [Docker Engine trên Ubuntu](https://docs.docker.com/engine/install/ubuntu/).
- [Shorebird pricing](https://shorebird.dev/pricing/).
- [Khởi tạo Shorebird](https://docs.shorebird.dev/code-push/initialize/).
- [Tạo release](https://docs.shorebird.dev/code-push/release/).
- [Tạo patch](https://docs.shorebird.dev/code-push/patch/).
- [Shorebird GitHub Actions](https://docs.shorebird.dev/code-push/ci/github/).
- [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions).
