# Kế hoạch 6 công việc trước production

- Ngày lập: 2026-11-01.
- Phạm vi: cửa hàng gia đình, offline-first, chấp nhận backend gián đoạn;
  ưu tiên dịch vụ miễn phí và thiết bị hiện có.
- Trạng thái: **cold-start fix đã deploy; lượt 11:20 fail read latency; đã bật cache DNS theo owner, soak mới chạy từ 12:21:36 ngày 04/11/2026 UTC+7 đến 19:00 cùng ngày; không phải 24h pass; chưa GO**.
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

### Trạng thái hiện tại sau deploy ngày 2026-11-03

- Live application source/image **`848d8be0127890746269957873314f54b6971cbe`**;
  deploy configuration `02623e1`, pilot runner `ab82b1d2aa67955fe1e761b9b6ffafb849c5c85c`.
  Khởi tạo thêm EF tracking/snapshot delegates bằng entity tạm trong bộ nhớ,
  DetectChanges rồi clear tracker; không SaveChanges, publish command hoặc tạo
  invoice trong DB. API invoice request/payload/queued envelope dùng generated
  JSON; test giữ nguyên bytes/hash v1/v2 và wire contract.
- Source commit sạch được full local **23/23**; cold **1023,49 / 661,77 /
  539,26 ms**. Release build 0 warning/error, regression **198 pass/4 opt-in
  skip**. [CI 41645651110](https://github.com/PanxiiPos/panxii-backend/actions/runs/41645651110)
  cho đúng `848d8be` đã success cả build-test và resilience; không retry/bỏ
  mẫu hoặc nâng ngưỡng 1500 ms.
- Image IDs đúng artifact local đã test, source/images transfer SHA-256 verified.
  Backup encrypted verified `manual/20261103T134454Z-c1942c4e6c30`.
  Đã activate chỉ API/Worker, giữ dependency container IDs/mounts/volumes,
  không migration/seed; giữ `d2dc524` và các image cũ để rollback.
  Attempt đầu tự rollback do probe nhầm public `/health/ready` (cố ý 404);
  đã dùng đúng private readiness như monitor và activate lại thành công.
  Không mở public readiness hoặc sửa Nginx để vượt probe.
- **Soak `20261103-135041` đã fail** lúc **21:31:37 ngày 03/11 UTC+7**,
  sau khoảng 40 phút 48 giây; `RemoteDisconnected` ở GET catalog.
  Run bắt đầu **20:50:49 ngày 03/11 UTC+7** (13:50:49Z);
  mốc dự kiến đủ 24 giờ ngày 04/11 không còn áp dụng.
  Dùng ngay hóa đơn đầu của run làm cold sample: **827,05 ms < 1500 ms**;
  không có warm-up invoice hoặc discarded sample. Preserve state/mẫu cũ,
  giữ 100 backlog invoice baseline và cấp UUID mới cho 100 invoice của lượt
  soak này, không dùng replay của hóa đơn cũ làm mẫu mới.
- Audit 13:54:54Z: **105 invoice/detail/completed receipt/change/published
  outbox** toàn tenant khớp; zero duplicate operations/sai line values;
  command/retry/DLQ đều rỗng. Cohort run mới ghi 101 (100 baseline + 1 mới),
  khác tổng lịch sử 105. Host monitor pass.
  Service `Type=oneshot` có `ActiveState=activating`, `SubState=start` và
  MainPID sống trong khi đang chạy 24 giờ; không coi nhãn này là lỗi.
- First cold sample đã pass nhưng lượt 24 giờ đã dừng, **chưa có 24h pass**.
  Vẫn phải audit cuối, UAT thiết bị thật và các gate GO/NO-GO bên dưới.

Evidence root-only: `coldstart-848d8be-prepared.json`,
`coldstart-848d8be-deployed.json`, `coldstart-848d8be-first-soak-sample.json`,
`coldstart-848d8be-database-audit.json`, `pilot-state.json`, `pilot-samples.jsonl`;
archives state/mẫu trước run có tên `*.before-848d8be-20261103-135041.*`.
Local full run: `panxii-resilience-b6d6237bb7f44f45a94b737b3d7cdd69`.
Transfer archives đã xóa; source/images/evidence và rollback giữ nguyên.

#### Khắc phục runner và lượt rút ngắn đến 19:00 ngày 04/11

- Đối chiếu incident: Nginx `keepalive_timeout 65`, runner nghỉ 60 giây cộng
  Docker stats/xử lý nên có thể dùng lại socket server đã đóng. Không có
  container restart/OOM hoặc API/Worker error marker trong cửa sổ lỗi.
  Probe thật trên VPS: socket không quản lý sau 70 giây gặp đúng
  `RemoteDisconnected`; runner sửa mở lại kết nối và search/pull thành công,
  TLS verification bật, không tạo invoice probe.
- Runner `1ce4b50`: chủ động reopen sau 50 giây idle; GET disconnect/reset/
  broken pipe được retry tối đa một lần có event rõ. HTTP lỗi/timeouts vẫn
  fail; không retry POST invoice/refresh/logout khi kết quả không rõ.
  Latency bao gồm TLS/reconnect/retry, transport events không bị bỏ khỏi
  evidence; log không có query/token/body. Logout lỗi không che lỗi gốc.
  **22 pilot test pass Windows và Python 3.10 trên VPS**;
  full deployment Python suite **36/36 pass trên Linux** (backup tests cần
  `fcntl`, không chạy được nguyên suite trên Windows).
- Audit trước restart: **107 invoice/detail/completed receipt/change/published
  outbox** toàn tenant khớp; zero duplicate/sai line values; command/retry/DLQ
  rỗng. Giữ nguyên failed state/samples của `20261103-135041`, không reset DB.
- Owner duyệt chạy **chỉ đến 19:00 ngày 04/11 UTC+7**. Run mới
  **`20261104-042043`** bắt đầu **11:20:49**, cutoff
  `2026-11-04T12:00:00Z`, khoảng **7 giờ 39 phút**.
  Giữ 5 clients, nhịp 864 giây/invoice và ngưỡng read p95 500 ms,
  pull/sync 1500 ms; dự kiến **32 invoice mới**, không ép tải 24h vào 8h.
  Giữ 100 baseline backlog commands, cấp UUID mới cho phần soak,
  không warm-up hoặc bỏ sample. API/Worker vẫn `848d8be`, mọi container ID,
  image/mount/start time không đổi: **chỉ cập nhật runner**, không deploy app.
  Lượt này sau đó **fail lúc 12:12:33** vì read latency; xem incident bên dưới,
  không dùng thời gian đã chạy để cộng vào một lượt pass.
- First invoice run mới **177,53 ms** trên process đang nóng, **không gọi là
  cold-start sample**. Đã quan sát 3 cửa sổ và 10 idle reconnect:
  max-window read p95 **450,86 ms**, pull p95 **29,61 ms**, không GET retry
  trong các cửa sổ đầu. PID sống, phase `soaking`.
- Workload ngừng sinh lúc cutoff rồi kiểm tra cuối/logout; cleanup có thể
  xong sau 19:00 một chút. Phase thành công dự kiến
  `completed-bounded-awaiting-database-audit`, **không phải đạt 24 giờ**.
  Còn final audit, UAT và các gate GO/NO-GO.
- Systemd drop-in có ngày:
  `/etc/systemd/system/panxii-pilot-soak.service.d/10-until-20261104.conf`
  truyền `--until 2026-11-04T19:00:00+07:00`; TimeoutStartSec giới hạn tới
  deadline cộng ba phút cho verification/logout. Cần bỏ/thay override này
  trước một lịch chạy khác; không tự chạy lại hoặc nối thời gian lượt lỗi.

Evidence root-only bổ sung: `pilot-idle-probe-20261104.json`/`.jsonl`,
`pilot-bounded-20261104-start.json`, `pilot-bounded-20261104-verified.json`,
state/mẫu hiện tại và archives
`pilot-state.failed-20261103-135041-archived-20261104-042043.json`,
`pilot-samples.failed-20261103-135041-archived-20261104-042043.jsonl`.
Runner cũ giữ tại `deploy/pilot.py.before-1ce4b50`; không chứa credentials.

#### Incident read latency và cache DNS ngày 04/11

- Run `20261104-042043` fail lúc **12:12:33 UTC+7**, sau khoảng 51 phút
  58 giây: **read p95 1344,62 ms > 500 ms**. Có 4 invoice mới, observed
  sync **177,53 / 470,71 / 707,71 / 501,71 ms**, đều dưới 1500 ms.
  Không phải lỗi socket cũ. Các catalog request lúc incident đều HTTP 200,
  Nginx request time **25-40 ms**. Probe phase timing sau đó: DNS thường
  **260-391 ms**, TLS **8-16 ms**; chưa có phase timing của request lỗi để
  khẳng định chính xác toàn bộ 1344,62 ms là DNS.
- VPS trước đó dùng trực tiếp `8.8.8.8/8.8.4.4`, không cache DNS. Owner duyệt
  bật `systemd-resolved` có sẵn, enable để tồn tại qua reboot, dùng stub
  `127.0.0.53`, upstream `8.8.4.4/8.8.8.8`, cache tôn trọng TTL.
  Đã kiểm chứng effective global DNS/domain và cache hits, không chỉ file
  cấu hình. Không ghim IP, sửa `/etc/hosts`, bỏ TLS hoặc loại DNS khỏi latency.
  [Runbook DNS/rollback](../infrastructure/operations_runbook.md#10-dns-host-chậm-trong-pilot).
- Probe sau sửa: cold DNS **44,95-267,64 ms**, cached **0,83-1,09 ms**;
  15 fresh TLS catalog searches **40,54-323,01 ms**; sau 70 giây idle/TTL
  hết hạn **353,11 ms**. Không tạo invoice probe; các mẫu đều dưới 500 ms.
  Điều này không bảo đảm DNS/mạng ngoài sẽ không chậm trong tương lai.
- Runner `ab82b1d` lưu `latency-window-failed` chứa đầy đủ read/pull/acceptance
  percentiles và violations trước khi raise, không mất cửa sổ lỗi khi clear
  metrics. **23 pilot test Windows / 37 deployment Python test Linux pass**,
  có boundary đúng 500 ms pass và 501 ms fail. Không đổi ngưỡng/retry policy.
- Audit trước chạy lại: **111 invoice/detail/completed receipt/change/published
  outbox** toàn tenant khớp, zero duplicate/sai line values; command/retry/DLQ
  rỗng. Archive nguyên failed state/mẫu, không reset DB; giữ 100 baseline
  commands và cấp UUID mới cho phần soak.
- Run mới **`20261104-052131`** bắt đầu **12:21:36 UTC+7**, vẫn cutoff
  **19:00 cùng ngày**, khoảng **6 giờ 38 phút**, dự kiến **28 invoice mới**
  theo nhịp 864 giây. First invoice **189,63 ms** trên process nóng, không
  coi là cold sample. Cửa sổ đầu read p95 **64,44 ms**, pull **13,12 ms**;
  host monitor pass. API/Worker và cả 7 container ID/image/mount/start time
  giữ nguyên. Chỉ host DNS và pilot runner thay đổi.
- Check **12:23:46 UTC+7**: phase `soaking`, PID sống, **3 cửa sổ đo**,
  max-window read p95 **361,59 ms**, pull **13,12 ms**; **10 idle reconnect,
  0 GET retry, 0 transport fail**. Runner checksum khớp, DNS service enabled,
  cutoff/OnFailure Discord vẫn gắn đúng. Đây chỉ là bằng chứng đang tiến triển,
  không phải kết quả cuối.
- Chưa có kết quả cuối hoặc 24h pass. Cutoff ngừng sinh tải, sau đó final
  verification/logout; vẫn cần final audit và các gate GO/NO-GO.

Evidence root-only: `pilot-dns-cache-20261104.json`,
`pilot-dns-probe-20261104.jsonl`, `pilot-dns-bounded-20261104-start.json`,
`pilot-dns-bounded-20261104-verified.json`;
archives `pilot-state.failed-20261104-042043-archived-20261104-052131.json`,
`pilot-samples.failed-20261104-042043-archived-20261104-052131.jsonl`.
Runner cũ: `deploy/pilot.py.before-ab82b1d`; resolver cũ:
`resolv.conf.before-dns-cache-20261104`. Không cộng dồn thời gian các lượt fail.

### Lịch sử bằng chứng ngày 2026-11-02 và điều tra trước deploy mới

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

Lượt sau sửa cursor bắt đầu **20:18:57 ngày 02/11/2026 UTC+7**, nhưng
**thất bại lúc 20:34:06 cùng ngày**, không đạt 24 giờ. Journal ghi observed
sync completion p95 **1923,80 ms > 1500 ms** ở hóa đơn soak thứ hai.
Mốc hoàn tất dự kiến ngày 03/11 không còn áp dụng. Có 14 cửa sổ đo hợp lệ:
max-window read p95 373,38 ms, pull p95 330,56 ms; đây không phải p95 toàn run.
Invoice đã commit trước khi script kiểm tra ngưỡng và lưu state nên state cũ
chỉ ghi 101; audit ngày 03/11 xác minh **102 invoice/details/completed receipts/
server changes/published DataCreated outbox**, không operation trùng, không
sai tổng hoặc line snapshot. Raw state/mẫu/journal được giữ; state hiện `failed`.

#### Điều tra và áp dụng cấu hình ngày 2026-11-03

- Incident proxy timings: POST 202 50 ms, operation receipt GET 28 ms, snapshot
  GET 17 ms; DB có completed receipt và published outbox. Tổng 95 ms phía Nginx
  không bao gồm toàn bộ DNS/TLS/client scheduling; không đủ để kết luận thời
  gian quan sát end-to-end tương đương worker commit latency.
- Runner của lượt 02/11 dùng `urllib.request.urlopen` mở kết nối mới mỗi request.
  Probe health cho thấy DNS 100-388 ms; diagnostic replay cùng một operation
  đã commit (không tạo invoice mới), 5 mẫu original mất 679,83-2164,33 ms;
  5 mẫu với HTTPS keep-alive có TLS verification mất 57,82-178,62 ms.
  Đây là bằng chứng chi phí kết nối góp phần vào lỗi đo, **không** là nghiệm thu
  invoice mới hoặc soak 24 giờ. Tại thời điểm diagnostic chưa deploy transport;
  phần cập nhật bên dưới ghi nhận lượt mới, không đổi số đo cũ thành pass.
- Owner đã sửa `.env`; đối chiếu chỉ thấy API/Worker thay
  `JWT_REFRESH_TOKEN_DAYS` từ 30 sang **300 ngày**. Backup manual mã hóa đã
  upload/download verified trước khi force-recreate **chỉ API/Worker**.
  Image IDs, volume mounts và các dependency container IDs giữ nguyên;
  không migration/build/seed/reset dữ liệu. Runtime env đã xác minh 300 ngày,
  public/private health, monitor và queue/DLQ pass. Policy này áp dụng khi cấp/
  rotate refresh token, không tự kéo dài hạn của token đã lưu.
- Sửa `PILOT_USERNAME/PILOT_PASSWORD` trong `.env` không thay account trong DB;
  probe ban đầu trả 401. Owner cho phép đồng bộ cả hai. Vì API hiện chưa có
  reset password, đã xác thực SuperAdmin qua HTTPS bằng hộp nhập riêng và
  thực hiện maintenance transaction: đúng một active TenantAdmin trong
  `PILOT-READINESS`, BCrypt tương thích, tăng version, thu hồi phiên cũ,
  ghi `TenantAdminCredentialsResetByMaintenance` audit. Login bằng đúng bộ
  credentials hiện tại của `.env` đã pass; phiên kiểm tra/SuperAdmin đã logout.
  Không ghi username/password/hash vào tài liệu hoặc audit payload.
- Sau maintenance, soak chưa restart tại thời điểm ghi nhận ban đầu. Runner mới
  bên dưới thay username-prefix guard bằng UUID của đúng tài khoản owner duyệt.

Evidence root-only trên VPS: `env-recreate-result-20261103.json`,
`pilot-account-maintenance-20261103.json`, `pilot-latency-diagnostic-20261103.json`,
`pilot-state.failed-latency-20261102.json` và `pilot-samples.jsonl`.

#### Lịch sử lượt lỗi và cold-start trước deploy `848d8be`

- Pilot runner `d24ed97` đã deploy: HTTPS keep-alive có TLS verification,
  pin đúng authorized user ID, lưu committed sample trước kiểm tra threshold,
  ghi state `failed` khi lỗi. 12 test Python pass Windows/Linux.
- Lượt `20261103-120145` bắt đầu 12:01:51Z (19:01:51 UTC+7), fail ngay hóa đơn
  mới đầu tiên: **3038,62 ms > 1500 ms**. Receipt thực sự còn 404 gần ba giây;
  đây không chỉ là overhead HTTPS. Không có lượt soak đủ 24 giờ.
- Owner chọn giữ soak dừng và cho phép khởi tạo EF/query **chỉ đọc** khi startup,
  không tạo hóa đơn warm-up hoặc bỏ mẫu. Bản `d2dc524` dùng composite ReadyToRun,
  bỏ detail joins khi chỉ kiểm tra invoice ID và khởi tạo query trước khi Worker
  subscribe. Full local 23/23, regression 196 pass/4 opt-in skip, cloud CI
  [41636263529](https://github.com/PanxiiPos/panxii-backend/actions/runs/41636263529)
  build-test/resilience pass.
- Đã backup encrypted verified rồi activate chỉ API/Worker `d2dc524`; dependency
  container IDs và volumes không đổi, không migration/seed. Worker startup mất
  2041,32 ms (ghi riêng). Hóa đơn mới đầu tiên trên VPS sau startup mất
  **1755,38 ms**, vẫn fail; không gọi đây là cold-start đã sửa xong.
- Follow-up source `6099f7a` chia sẻ startup reads cho API/Worker, khởi tạo query
  tenant/session và dùng generated JSON cho invoice command/hash/realtime.
  Test so khớp exact JSON với serializer cũ, giữ hash PascalCase v1/v2 và event
  camelCase. Regression 198 pass/4 opt-in skip; full local 23/23 pass, nhưng
  cloud CI [41639847324](https://github.com/PanxiiPos/panxii-backend/actions/runs/41639847324)
  có build-test pass và resilience **22 pass/1 fail**: ba mẫu cold API/Worker
  **2052,85 / 1833,21 / 1771,00 ms**. Không retry/bỏ mẫu/nới ngưỡng để báo xanh.
  Candidate source/images đã chuẩn bị nhưng **chưa activate**; live vẫn
  `d2dc5244f8cfccc1c94e49a2cc1d69508e69b5c9`.
- Audit 13:07:07Z: 104 invoice/detail/completed receipt/change/published
  DataCreated outbox của invoice khớp, zero duplicate operation groups hoặc
  sai line values. Có một invoice diagnostic mới ngoài cohort soak đã dừng;
  giữ lịch sử và audit riêng, không dùng replay của nó làm mẫu cold mới.
  Monitor pass, disk 21% (~38 GiB available), transfer archives đã xóa.
  `panxii-pilot-soak.service` MainPID=0; state lượt trước vẫn `failed`.

Evidence root-only bổ sung: `coldstart-deploy-before-20261103.json`,
`coldstart-deploy-result-20261103.json`, `coldstart-first-invoice-20261103.json`,
`coldstart-database-audit-20261103.json`. Còn phải profiling phần first-write/
first-result materialization, xác minh ngưỡng trên VPS và CI trước khi chốt fix
hoặc bắt đầu lại một lượt soak mới.

#### Phân rã latency trên stack cô lập, 2026-11-03

- Không restart/deploy production hoặc chạy lại soak. Kiểm tra VPS chỉ đọc
  không thấy quota CPU throttling ở API/Worker; RAM available khoảng 2,3 GiB.
  Đây là snapshot hiện tại, không chứng minh host không bị contention lúc lỗi.
- Diagnostic run `panxii-resilience-752bb31398a54314ab9f2755e4171e27`:
  cold **2625,96 / 1859,79 / 1627,73 ms**; hóa đơn mới thứ hai cùng process
  **375,6 / 314,4 / 343,6 ms**. POST cold 505–1394 ms, so với 37–72 ms
  ở hóa đơn thứ hai. Mỗi cold sample vẫn bị kiểm ngưỡng 1500 ms; không bỏ mẫu.
- Không có EF query compilation sau lúc submit. Tổng SQL Worker quan sát
  khoảng 19–21 ms mỗi cold sample (có cả outbox polling xen kẽ), trong khi
  lần SaveChanges đầu mất 178–233 ms so với 6–11 ms khi process đã nóng.
  DetectChanges đầu 113–143 ms so với dưới 1 ms; đây là phần nằm trong
  SaveChanges, **không cộng hai số thành tổng**. Log còn chỉ ra chi phí
  materialization/tracking thực tế khi query lần đầu trả dữ liệu.
  Startup probe tenant ngẫu nhiên trả rỗng chỉ khởi tạo query shape,
  không chạy hết đường materialization/tracking/ghi đầu tiên.
- Đo riêng publisher trong run
  `panxii-resilience-418ff2dc8eee4abbb956b847d8d23182`:
  cold **1340,72 / 1523,98 / 1617,09 ms**; hóa đơn mới thứ hai
  **294,32 / 428,77 / 294,68 ms**. Mở connection/channel/confirm mode
  chỉ 6–8 ms; serialize queued command khoảng 1,5–1,7 ms.
  Topology 11–80 ms, publish/confirm 11–75 ms; một confirm ở hóa đơn thứ hai
  còn mất 123 ms. Vì vậy không quy toàn bộ POST chậm cho lazy RabbitMQ setup.
- Kết luận đã xác minh: chi phí first-use còn ở API và EF khi có dữ liệu/ghi,
  không chỉ SQL hoặc query translation. Chưa có CPU/JIT trace để phân bổ hết
  phần API/framework còn lại; không khẳng định chính xác bao nhiêu ms là JIT.
  Receipt wait có cả polling 200 ms và HTTP, không phải pure Worker CPU.
  Debug logging ảnh hưởng số đo; các lượt này chỉ phục vụ diagnosis,
  không thay thế CI/VPS acceptance. Live vẫn `d2dc524`, soak vẫn dừng.
- Kiểm tra lại logging mặc định bằng `-ColdStartOnly` trên source diagnostic
  chưa commit: run `panxii-resilience-32d4bced0eb54df2884c77910fedc844`,
  1/1 test pass, ba cold sample **1249,46 / 1350,36 / 1388,38 ms**,
  không có comparison/warm-up invoice, cleanup hoàn tất. Đây là local pass,
  không phải fix đã được xác nhận trên CI hoặc VPS.

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
