# Kế hoạch 6 công việc trước production

- Ngày lập: 2026-11-01.
- Phạm vi: cửa hàng gia đình, offline-first, chấp nhận backend gián đoạn;
  ưu tiên dịch vụ miễn phí và thiết bị hiện có.
- Trạng thái: **.NET 10/APK production/private admin đã triển khai; soak đang chạy; chưa GO**.
- Không tự nâng gói trả phí, dùng credit trial như miễn phí lâu dài hoặc
  thay đổi dữ liệu development/production để chuẩn bị release.

## 1. Điểm xuất phát và nguyên tắc

Đã có bằng chứng local: backend resilience 22/22 trên hai stack; Android
development 3+8 checkpoint; 8 invoice không trùng; backup development restore
khớp 18 bảng. Đây không phải bằng chứng hạ tầng production đang hoạt động.
Xem [ma trận kiểm thử](../testing/risk_based_testing_spec.md).

Giai đoạn đầu ngày 2026-11-02 đã dựng VPS private: API/Worker Production, MySQL/RabbitMQ,
17 migrations và duy nhất một SuperAdmin; health, login/platform/logout,
bootstrap chống ghi đè và restart API/Worker đã pass. Các bảng nghiệp vụ trống.
Secrets chỉ trên VPS `.env` quyền 600; SSH key-only, origin API chỉ loopback.
Sau đó đã triển khai Nginx public cho `https://panxii.duckdns.org`, backup mã hóa
Google Drive và monitoring/Discord như bảng dưới. Pipeline deploy, signing release
và Shorebird patch/rollback chưa nghiệm thu; **chưa GO production**. Oracle Always Free đã
bị loại, VPS thuê thay thế. Source đang ở branch feature; không đồng nghĩa
đã merge. Xem [trạng thái VPS](../infrastructure/internal_free_deployment.md).

### Bằng chứng hiện tại ngày 2026-11-02 (thay thế trạng thái giai đoạn đầu bên dưới)

- Live application source/image `720fb2a5719a01c2228685ffa145bbe176fe2d69`;
  deploy configuration `02623e1a1ef05bdc84ee171be26cb894e6227645`;
  sửa pilot cursor/preflight trên branch tại `04325ee`.
- Bảy project target `net10.0`, SDK `10.0.403`; API/Worker runtime `10.0.14`.
  EF `8.0.33`/Pomelo `8.0.3` tạm giữ theo owner; MySqlConnector `2.6.2`.
  .NET 10 LTS EOL 2028-11-14, nhưng EF 8 EOL 2026-11-10 vẫn là blocker.
  Chưa xác minh stable provider EF 10 phù hợp; cần chốt strategy và review
  lifecycle MySQL/RabbitMQ trước GO, không tự nâng major trên volume.
- Release build 0 warning/error; regression 196 pass/4 opt-in integration skip;
  NuGet transitive audit không có known advisory. Python backup 14 pass,
  pilot 6 pass Windows/Linux. Docker resilience 22/22 pass sau diễn tập
  .NET 10 -> đúng image VPS .NET 8 trước -> .NET 10, cùng DB, invoice read/replay
  không trùng, không migration. Đây là rollback runtime cô lập, không phải
  diễn tập restore hoặc rollback schema production.
- [Backend CI run 41450422464](https://github.com/PanxiiPos/panxii-backend/actions/runs/41450422464)
  tại `67d00b7` xanh cả `build-test` và `resilience`; không tự deploy.
  Required branch checks/merge gate chưa xác nhận; branch feature chưa merge.
  Run mới `41452714399` cho `02623e1` đang chờ/chạy khi ghi nhận; không suy
  trạng thái xanh của `67d00b7` sang SHA mới.
- Swagger chỉ listener loopback 8081, JSON có 46 paths; public Swagger/readiness
  trả 404 kể cả spoof forwarded headers. Dozzle 11.4.0 qua socket proxy read-only,
  không public, không shell/actions/analytics. Tunnel workstation đã HTTP 200:
  `http://localhost:18081/swagger`, `http://localhost:19999`.
  UI Dozzle đã thấy đủ 7 containers và log API; proxy GET container list pass,
  fake mutation/exec endpoints trả 403. Tắt registry image checks/auto-update
  thay vì mở rộng quyền proxy. Dozzle dùng quyền SSH key, không dùng tài khoản ứng dụng. Owner xác nhận
  đăng nhập SuperAdmin trên Swagger thành công; **chưa có UI SuperAdmin mobile**,
  không dùng màn hình quản trị mobile làm gate nghiệm thu.
- APK ký production `0.1.4+5`, package `com.panxiipos.panxii_mobile.production`,
  cài/chạy trên một Samsung SM-A336E, song song app debug không xóa dữ liệu.
  [APK HTTPS](https://panxii.duckdns.org/downloads/panxii-0.1.4-production.apk)
  đã đối chiếu hash; minimum native `0.1.0`, latest `0.1.4`.
  Shorebird patch 1 đã tải/verify/apply và khởi động được; rollback thực tế về
  native base cũng pass. Đã roll forward patch trên server, nhưng chưa xác nhận
  thiết bị nhận lại patch; không báo roll-forward end-to-end pass.
  File khôi phục khóa ký mã hóa ngoài repo; owner xác nhận giữ mật khẩu.

#### Capacity/pilot được duyệt và tiến độ

Owner duyệt 5 clients, 200 hóa đơn/ngày, backlog 100, 2000 biến thể;
tenant synthetic `PILOT-READINESS`, không tạo dữ liệu cửa hàng thật.
**Database hiện không còn trống nghiệp vụ**: test tenant/2000 products và
2000 variants được giữ để audit; không seed lại/xóa lịch sử invoice.

| Phép đo prepare | Kết quả | Ngưỡng |
|---|---|---|
| 100 invoice commit + replay + snapshot/receipt | 18,84 giây | <= 60 giây |
| Read p50/p95/p99 (421 samples) | 111,71 / 234,72 / 432,76 ms | p95 <= 500 ms |
| Pull p50/p95/p99 (55 samples) | 192,88 / 287,73 / 386,10 ms | p95 <= 1500 ms |
| POST acceptance p95 (200 samples) | 251,68 ms | Không thay cho durable completion |

Lượt soak đầu dừng trước mẫu đầu vì script search thiếu `cursor=`, HTTP 400;
không phải 24 giờ pass. Đã sửa và test, giữ evidence lượt lỗi, audit DB theo
UUID commands: 101 invoice/details/completed receipts/server changes/published
DataCreated outbox khớp, 0 operation trùng, 0 sai tổng tiền. Invoice 101 đã
commit trước lỗi; restart dùng lại đúng operation/invoice IDs, không thêm bản sao.
Không scope audit theo tiền tố số hóa đơn client vì server chuẩn hóa số.

Lượt mới `panxii-pilot-soak.service` đã xác minh PID sống và có mẫu mới:
bắt đầu **20:18:57 ngày 02/11/2026 UTC+7**, dự kiến đủ 86400 giây lúc
**20:18:57 ngày 03/11/2026 UTC+7**. Mẫu 20:21:09 có 101 invoice,
read p95 307,79 ms, pull p95 302,42 ms, observed sync p95 254,10 ms;
queue ready/unacked và DLQ đều 0. Read/pull kiểm tra từng cửa sổ khoảng một
phút; observed sync p95 tích lũy, ngưỡng 1500 ms. Bằng chứng đầu run không
thay cho đủ 24 giờ, invoice mới sau refresh token và final DB/outbox/DLQ audit.
Systemd oneshot ở `activating/start` trong khi chạy là dự kiến, không phải lỗi.

Năm clients HTTPS chạy từ VPS, **không phải năm điện thoại hoặc mạng ngoài**.
Còn gate thiết bị thật TenantAdmin: catalog, invoice/offline/restart,
sync/pull/SignalR. In Canon thật được owner loại khỏi gate bắt buộc ngày đầu.
OTA vẫn bắt buộc ngày đầu.

**NO-GO còn lại:** đủ soak và audit cuối; mobile pilot nghiệp vụ; hai lịch
backup tự động thật; independent replacement-VPS/key/API restore và RTO;
availability monitor ngoài VPS; provider/image lifecycle; CI merge gates;
owner nghiệm thu/GO và rotation credentials từng lộ trong chat.

### Lịch sử giai đoạn đầu ngày 2026-11-02 (không phải trạng thái hiện tại)

Source/image VPS: `3f171a2181b5cc27ef587581f5386a4047455b54`, branch
`feature/production-vps`; API runtime `8.0.33`. Kiểm tra cuối sau reboot/deploy:
0 gói OS chờ update, không có systemd unit failed, 18 bảng vẫn chỉ có một
admin cùng metadata/default settings, không có dữ liệu nghiệp vụ/session/audit.
Probe query-token không xuất hiện trong cả log Nginx/API; runtime containers
không nhận OAuth/crypt/webhook hoặc root/bootstrap secrets. Giữ image API/Worker
trước để rollback; chưa nghiệm thu diễn tập rollback staging.

Đối chiếu production env ngày 2026-11-02: file thật ở `/opt/panxii/.env`
trên VPS, root:root 600; đủ 35 biến cấu hình không-bootstrap trong
[template backend](../../backend/deploy/production.env.example). Đã khai báo
và apply vào API/Worker thời hạn access token 15 phút, refresh token 30 ngày;
policy minimum/latest app giữ `0.1.0`. Kiểm tra giá trị runtime, HTTPS
`/api/v1/config/version`, dependency readiness và monitor đều pass sau recreate.
Không đổi credentials, image ứng dụng hoặc database; backup mã hóa trước
thay đổi đã xác minh. File template không chứa secret và không thay thế `.env`
development. URL APK/patch chưa có là gate release còn thiếu, không tự điền
URL giả hoặc ép nâng phiên bản chưa được ký/nghiệm thu.

| Công việc | Đã kiểm chứng | Còn thiếu để nghiệm thu toàn gate |
|---|---|---|
| 1. Hạ tầng | API/Worker/MySQL/RabbitMQ/Nginx x64; secrets VPS-only, origin private; cập nhật 326 gói OS và 17 gói mới, boot kernel `5.15.0-202-generic`, SSH/HTTPS trở lại | Review lifecycle/patch các runtime/image, smoke nghiệp vụ và hồ sơ quota/capacity |
| 2. Backup | Rclone crypt lên Google Drive; lịch 06/12/21 UTC+7; giữ 6 bản scheduled đã xác minh, xóa vĩnh viễn bản cũ sau thành công; manual riêng; upload/download/decrypt pass; restore MySQL `network none` khớp fingerprint dữ liệu 18 bảng (~33 giây); khóa khôi phục xuất thành file mã hóa ngoài VPS, owner xác nhận giữ mật khẩu | Hai lịch tự động liên tiếp thực sự, restore bằng file recovery trên môi trường thay thế, API/login/sync/pull/reverse và RTO đầy đủ; thời gian ~33 giây không phải RTO cam kết |
| 3. HTTPS | DuckDNS trỏ đúng VPS, chứng chỉ Let's Encrypt đúng hostname đến 2027-01-31; HTTP 308; renewal dry-run pass; chỉ mở 22/80/443; trusted proxy một hop, chống giả IP vượt auth rate limit, Swagger/dependency readiness không public; Nginx hỗ trợ WebSocket/DNS recreate | Login/sync/pull/SignalR có xác thực trên thiết bị cửa hàng; kiểm tra giới hạn batch thực tế |
| 4. APK/OTA | Chưa thay release/signing; APK debug cũ không được coi là production | Toàn bộ gate release signing, phân phối HTTPS và OTA theo phạm vi đã chốt |
| 5. Vận hành | Host monitor 5 phút: health, services, disk, tuổi backup, hạn TLS; OnFailure backup/TLS/monitor gửi Discord, inject lỗi và owner xác nhận nhận; runbook/rollback private có sẵn | Monitor độc lập ngoài VPS, queue/outbox/latency/RAM, CI gates, deploy/rollback staging và người vận hành nghiệm thu |
| 6. Capacity/pilot | Không tạo dữ liệu demo/cửa hàng trên database mới; chưa chạy stress vào VPS | Chốt workload, load/soak, thiết bị pilot, Canon nếu dùng ngày đầu và quyết định GO |

Kiểm thử thay đổi: 27 proxy/authorization tests .NET pass; API Release build
0 warning/error. Test project còn warning EF Relational đã tồn tại từ trước.
14 test Python backup pass, gồm dump rỗng/truncated/lỗi, upload/download/marker
lỗi không prune, chống clock rollback và giữ đúng 6 bản, xóa không qua Drive Trash.
Các test retention dùng storage giả; không tạo/xóa dữ liệu thử trong Drive thật.

**Blocker lifecycle:** metadata chính thức của Microsoft ghi .NET 8 EOL
**2026-11-10**; cần kế hoạch nâng runtime/framework và nghiệm thu tương thích
trước vận hành lâu dài, không coi pin image là còn được hỗ trợ.
Xem [release metadata](https://raw.githubusercontent.com/dotnet/core/main/release-notes/releases-index.json).
Lifecycle MySQL/RabbitMQ cũng phải được đối chiếu vendor trước GO; chưa tự
nâng major database/broker trên volume hiện có trong đợt cấu hình này.

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

1. Nhà cung cấp VPS, backup và hostname **đã chốt**: thuevpsgiare.vn gói VPS SSD C4; Google Drive qua Rclone; `panxii.duckdns.org`.
2. RPO/retention **đã chốt** (3 lượt/ngày, giữ 6 bản mới nhất, chấp nhận mất tới ~9 giờ). Còn phải chốt RTO sau drill và người giữ recovery key.
3. Số thiết bị/tải/backlog cần nghiệm thu; ngưỡng latency và soak duration.
4. OTA và in Canon có bắt buộc ngày đầu không; thiết bị pilot/kênh tải APK.
5. Kênh cảnh báo **đã chốt Discord**, owner đã xác nhận nhận alert thử; còn người quyết định rollback/GO.

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
