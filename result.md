# Tổng kết dự án Panxii theo 6 giai đoạn SDLC

**Cập nhật:** 06/11/2026
**Phạm vi tổng kết:** nền tảng POS SaaS Panxii, gồm backend, ứng dụng mobile, dữ liệu, đồng bộ và triển khai nội bộ. Nội dung phản ánh trạng thái trong tài liệu dự án cập nhật đến ngày 05/11/2026.

## Tổng quan

Panxii được xây dựng để giúp cửa hàng tiếp tục bán hàng khi kết nối không ổn định, đồng thời đồng bộ giao dịch an toàn khi có mạng. Hệ thống hướng tới mô hình SaaS nhiều cửa hàng (multi-tenant), phân quyền theo vai trò và lưu trữ dữ liệu cục bộ được mã hóa trên thiết bị.

Phần lõi hiện bao gồm quản lý tenant và tài khoản, xác thực, sản phẩm và biến thể, danh mục, hóa đơn, cấu hình cửa hàng, lưu hóa đơn offline và đồng bộ qua backend. Backend đã được triển khai trên VPS với .NET 10; ứng dụng Android đã có bản phát hành nội bộ 0.1.6+8 và bản vá OTA đã được xác nhận hoạt động trên thiết bị thật.

Kết quả này cho thấy Panxii đã đi qua cả phát triển lẫn triển khai thực tế. Tuy nhiên, **chưa có đủ bằng chứng để tuyên bố toàn bộ hệ thống đã đạt GO production**: kịch bản phục hồi dữ liệu khi server gặp sự cố được để lại xử lý sau, một số kiểm tra nghiệm thu và các cổng vận hành vẫn còn chờ hoàn tất.

## Công nghệ và phiên bản

Các phiên bản runtime/dependency dưới đây được đối chiếu từ cấu hình dự án và trạng thái triển khai được ghi nhận. Phiên bản công cụ trong workspace được tách riêng với phiên bản runtime trên VPS; những thành phần chưa có version pin rõ ràng được ghi là chưa xác định thay vì suy đoán.

| Nhóm | Công nghệ | Phiên bản ghi nhận | Phạm vi/ghi chú |
|---|---|---|---|
| Backend runtime | .NET / ASP.NET Core | .NET runtime 10.0.14; SDK workspace 10.0.403 | VPS và công cụ phát triển là hai thông tin riêng |
| Kiến trúc backend | Clean Architecture | — | Domain, Application, Infrastructure, Web API |
| ORM | Entity Framework Core | 10.0.14 | Backend đang deploy |
| MySQL provider | MySql.EntityFrameworkCore | 10.0.12 | Provider chính thức đang deploy |
| Database | MySQL | 8.0 | Local image `mysql:8.0`; patch version VPS không ghi nhận |
| Messaging | RabbitMQ | 3.13 (image cấu hình local) | Version image trên VPS chưa được xác nhận trong tài liệu |
| RabbitMQ client | RabbitMQ.Client | 6.8.1 | Backend |
| API / realtime | ASP.NET Core Web API, SignalR | .NET 10.0.14; Flutter client `signalr_netcore` 1.4.4 | SignalR dùng cho thông báo thay đổi theo tenant |
| Mobile framework | Flutter | 3.47.7 | SDK trong workspace |
| Mobile language | Dart | 3.13.5 | SDK workspace; constraint dự án `^3.13.5` |
| Mobile release | Panxii Android | 0.1.6+8 | APK production nội bộ |
| Mobile local database | SQLite/SQLCipher (`sqflite_sqlcipher`) | 3.4.1 | Version package Flutter; version thư viện native SQLCipher chưa ghi nhận |
| State management | `flutter_bloc` | 9.1.1 | Mobile |
| Localization | `easy_localization` | 3.0.8 | Mobile |
| UI/theme | `flex_color_scheme`, `forui`, `hugeicons` | 8.4.0, 0.27.6, 1.2.0 | Mobile |
| OTA | `shorebird_code_push` | 2.0.7 | Bản vá OTA đã được xác nhận áp dụng trên thiết bị |
| Hệ điều hành VPS | Ubuntu Server | 22.04 x64 | VPS nội bộ |
| Container tooling | Docker Compose | 5.5.1 | Workspace hiện tại; version Docker trên VPS không ghi nhận |
| Reverse proxy | Nginx | Chưa xác định | Đã cấu hình HTTPS; version không được pin trong bằng chứng |
| Backup | Rclone | 1.75.2 | Mã hóa và lưu backup trên Google Drive |
| Quản lý log | Dozzle | 11.4.0 | Truy cập nội bộ qua SSH tunnel |
| In hóa đơn | Windows Printer Bridge | Chưa xác định | Đã xác nhận in vật lý thành công |

## 1. Phân tích yêu cầu và nghiệp vụ

Ngay từ đầu, yêu cầu sản phẩm tập trung vào những tình huống quan trọng tại quầy: lập hóa đơn nhanh, không mất dữ liệu khi mất mạng, kiểm soát quyền và giữ lịch sử giao dịch có thể truy vết. Phạm vi người dùng được chia thành ba vai trò: SuperAdmin quản trị nền tảng, TenantAdmin quản lý cửa hàng và Staff thực hiện nghiệp vụ bán hàng.

Các quy tắc cốt lõi đã được xác định rõ. Dữ liệu cửa hàng phải được cách ly theo tenant; nhân viên không được tự thay đổi giá hay cấu hình; hóa đơn đã đồng bộ không bị sửa hoặc xóa trực tiếp mà được xử lý bằng bút toán đảo và hóa đơn điều chỉnh. Việc retry đồng bộ phải an toàn, không tạo giao dịch trùng, còn dữ liệu đang chờ đồng bộ phải được giữ lại trên thiết bị.

Phạm vi hiện tại cũng phân biệt rõ tính năng cốt lõi với phần mở rộng. POS, danh mục, sản phẩm, hóa đơn và đồng bộ là trọng tâm; Bluetooth, một số nghiệp vụ khách hàng trên mobile, kho, chấm công và các hình thức thanh toán mở rộng chưa được xem là các module đã hoàn tất. Các quy tắc này được tập hợp trong [business_rule.md](./business_rule.md) và [project_overview.md](./project_overview.md).

## 2. Thiết kế kiến trúc và giải pháp

Panxii chọn kiến trúc Clean Architecture cho backend ASP.NET Core, với các lớp Domain, Application, Infrastructure và Web API. Cơ sở dữ liệu server sử dụng MySQL 8; Flutter và BLoC được dùng cho mobile, với SQLite/SQLCipher làm nơi lưu dữ liệu cục bộ. RabbitMQ đảm nhiệm xử lý bất đồng bộ, còn SignalR thông báo thay đổi đến các thiết bị trong cùng tenant.

Luồng đồng bộ được thiết kế để tách phản hồi tiếp nhận khỏi xác nhận hoàn tất: API có thể trả về trạng thái hàng đợi, worker xử lý command theo cách idempotent, rồi ghi hóa đơn, sequence thay đổi và outbox trong cùng transaction. SignalR chuyển snapshot sau commit; nếu thiết bị bỏ lỡ thông báo, API pull theo sequence giúp khôi phục thay đổi.

Thiết kế cũng bao quát JWT và RBAC, refresh token, tenant isolation, error envelope thống nhất, correlation ID, đa ngôn ngữ Việt/Anh, cấu hình feature flags và khả năng tương thích API giữa mobile với backend. Các hợp đồng API, schema, luồng dữ liệu và quyết định kiến trúc được mô tả trong [system_architecture.md](./architecture/system_architecture.md), [api_contract.md](./api/api_contract.md) và [database_design.md](./database/database_design.md).

## 3. Phát triển và tích hợp

### Backend

Backend đã có luồng xác thực và làm mới phiên, quản lý vòng đời tenant và người dùng quản trị, global feature flags, cùng các API phục vụ sản phẩm, danh mục, khách hàng, hóa đơn, cấu hình cửa hàng và đồng bộ. TenantAdmin có thể quản lý sản phẩm/biến thể và nhân viên; Staff có các quyền đọc và nghiệp vụ bán hàng phù hợp với vai trò. Hóa đơn reverse/correction được xử lý theo nguyên tắc giữ lịch sử.

API áp dụng response envelope, error code và correlation ID thống nhất; kiểm soát role và tenant ở phía server. Các thao tác ghi quan trọng được xử lý qua worker và RabbitMQ thay vì coi phản hồi tiếp nhận là giao dịch đã hoàn tất. Runtime backend được triển khai với .NET 10.0.14, EF Core 10.0.14 và MySql.EntityFrameworkCore 10.0.12.

### Mobile

Ứng dụng Flutter đã có luồng đăng nhập, lưu phiên an toàn, gia hạn token, theo dõi kết nối và giao diện cho sản phẩm, danh mục, lập hóa đơn, danh sách/chi tiết hóa đơn cùng cấu hình tenant. Local database sử dụng SQLCipher; dữ liệu sản phẩm, cấu hình và hóa đơn có cache hoặc trạng thái local phù hợp cho trải nghiệm offline. Hóa đơn có thể được giữ trong outbox và đồng bộ khi kết nối trở lại.

Giao diện phân biệt trạng thái nghiệp vụ với trạng thái đồng bộ, hỗ trợ tìm kiếm sản phẩm và biến thể, nhập liệu hóa đơn, thông báo lỗi và bản địa hóa. Phần in A4 qua Windows Printer Bridge đã được xác nhận in thành công trên máy in vật lý tại cửa hàng.

### Dữ liệu và cấu hình

Mô hình server bao gồm tenant, người dùng, thiết bị, sản phẩm/biến thể, danh mục, khách hàng, hóa đơn, operation sync, outbox và change feed. Các quan hệ tenant-scoped được bảo vệ bằng tenant context và các ràng buộc dữ liệu. Trên mobile, các lần nâng schema SQLCipher đã bổ sung cache cấu hình, danh mục, dữ liệu catalog và hóa đơn mà vẫn giữ dữ liệu chờ đồng bộ.

## 4. Kiểm thử và bảo đảm chất lượng

Kiểm thử được tổ chức theo rủi ro: rò rỉ dữ liệu giữa tenant, sai thứ tự đồng bộ, lỗi SQLite/máy in và thay đổi API không tương thích. Các lớp kiểm tra gồm unit test, integration test, API contract, kiểm thử offline/restart, resilience với Docker/MySQL/RabbitMQ và xác minh trên Android development.

### Kết quả kiểm thử

| Bộ kiểm thử | Kết quả chính xác được ghi nhận | Đánh giá |
|---|---:|---|
| Backend resilience gate | 23/23 pass, 0 skip | Pass; có cold-start/restart và kiểm tra không ghi dữ liệu khi khởi động |
| MySQL provider compatibility | 6/6 pass, 0 skip | Pass; CI chạy build-test, provider compatibility và resilience trên commit được nghiệm thu |
| Backend regression (`dotnet test Panxii.sln`, 06/11/2026) | 230 pass, 0 fail, 25 skip (255 total) | Toàn bộ test được thực thi đều pass; 25 test opt-in/fixture không chạy trong lệnh mặc định |
| Mobile unit/widget (`flutter test`, 06/11/2026) | 299 pass, 1 fail, 1 skip (301 total) | Chưa xanh toàn bộ; lỗi ở test invoice composer trên màn hình 320×640 |
| Mobile development resilience | 3 + 8 checkpoint; 8 hóa đơn được đồng bộ, không tạo trùng ở server | Pass trong môi trường development; không thay thế UAT cửa hàng |
| In hóa đơn vật lý | Đã xác nhận in thành công tại cửa hàng | Pass theo xác nhận của chủ dự án |

Số liệu backend regression **200 pass, 5 skip** trong báo cáo trước đã được thay bằng lượt chạy đầy đủ mới nhất nêu trên. Backend và mobile dùng runner khác nhau; tổng số test được tìm thấy không phải coverage theo use case. Mobile integration test suite không nằm trong lệnh `flutter test`; các checkpoint resilience được báo cáo riêng. Test mobile thất bại là `quantity and services are directly editable with keyboard on narrow large-text screen` (`mobile/test/features/invoices/invoice_composer_test.dart`): nút lưu nằm ngoài viewport 320×640, thao tác tap không thực hiện được và test gặp null check. Không tính test này là pass.

### Số liệu soak/capacity

Ngưỡng được giữ xuyên suốt các lượt: **read p95 ≤ 500 ms**, **pull p95 ≤ 1.500 ms**, **invoice sync p95/max ≤ 1.500 ms**. Thời gian read bao gồm kết nối mạng/TLS và retry GET được phép; POST invoice không tự retry khi kết quả không rõ.

| Run | Thời lượng/kết thúc | Read | Pull | Đồng bộ hóa đơn | Kết quả |
|---|---|---:|---:|---:|---|
| `20261103-135041` | Khoảng 40 phút 48 giây; dừng 03/11 | Không có percentile cuối; gặp `RemoteDisconnected` ở GET catalog | — | Cold invoice đầu tiên 827,05 ms | Fail do ngắt kết nối; không phải lượt 24 giờ |
| `20261104-042043` | Dừng sau khoảng 51 phút 58 giây | p95 **1.344,62 ms**, vượt ngưỡng 500 ms | Không có p95 được ghi nhận cho cửa sổ lỗi | 4 hóa đơn: **177,53 / 470,71 / 707,71 / 501,71 ms**, tất cả dưới 1.500 ms | Fail do read latency |
| `20261104-052131` | Dừng sau khoảng 4 giờ 36 phút 56 giây | p95 cửa sổ lỗi **1.316,50 ms**, vượt ngưỡng; p50 read **41,98 ms** | p95 cửa sổ lỗi **11,21 ms** | 20 hóa đơn mới hoàn tất; sync lớn nhất **530,60 ms** | Fail do read latency |
| `20261104-100534` | **1 giờ 54 phút 26 giây**, hoàn tất 04/11 lúc 19:00:04 UTC+7 | Max-window read p95 **94,05 ms** | Max-window pull p95 **46,52 ms** | Sync p95/max được ghi nhận **300,57 ms** | **109 cửa sổ**; 8/8 hóa đơn hoàn tất; 0 reconnect, 0 GET retry, 0 transport fail; audit 139 bản ghi khớp, queue/retry/DLQ rỗng |

Chủ dự án chấp nhận lượt `20261104-100534` và audit cuối để đóng gate synthetic soak/correctness, đồng thời miễn thời lượng 24 giờ. Đây là **bounded soak pass theo ngoại lệ đã duyệt**, không phải bằng chứng chạy liên tục 24 giờ. Hai lượt read-latency fail vẫn được giữ trong lịch sử; không cộng gộp thời gian các lượt lỗi với lượt pass.

### Hai thiết bị sửa cùng dữ liệu khi một thiết bị offline

**Cơ chế chống ghi đè đã có, nhưng chính sách giải quyết xung đột và kịch bản end-to-end này chưa hoàn tất.** Product/catalog update gửi `expectedVersion`; server so sánh với version hiện tại và từ chối version cũ bằng `ERR_VERSION_CONFLICT`. Client lưu command offline, khi online sẽ gửi lại; nếu server đã có version mới hơn, operation được đánh dấu `failed` và giao diện báo lỗi thay vì âm thầm ghi đè dữ liệu server. Thay đổi local chưa được tự động hòa trộn vào bản server.

Ví dụ: thiết bị A đang offline sửa sản phẩm ở version 4; thiết bị B online sửa và lưu trước, làm server lên version 5; khi A kết nối lại, lệnh của A vẫn mang `expectedVersion: 4` nên bị từ chối xung đột. Người dùng cần xem dữ liệu mới nhất rồi chủ động bỏ hoặc nhập lại chỉnh sửa. Các test hiện có kiểm tra version conflict và đồng bộ theo version, nhưng tài liệu chưa ghi nhận một bài E2E cụ thể bao trọn trình tự hai thiết bị này; chính sách merge khi hai thiết bị cùng sửa product/customer vẫn là quyết định cần chốt trước khi coi luồng đó hoàn thiện.

Chiến lược và phạm vi kiểm thử được mô tả trong [testing_strategy.md](./testing/testing_strategy.md) và [risk_based_testing_spec.md](./testing/risk_based_testing_spec.md).

## 5. Triển khai và phát hành

Backend đã được triển khai lên VPS và nâng lên runtime .NET 10/EF Core 10. Các lần triển khai được ghi nhận có kiểm tra health/readiness, worker và backup trước thao tác dữ liệu; việc cập nhật ứng dụng giữ nguyên các volume và dịch vụ phụ trợ cần thiết. HTTPS qua Nginx đã được cấu hình; các cổng database và message broker không được mở công khai.

Ứng dụng Android nội bộ đã có APK production ký đúng khóa, phiên bản **0.1.6+8**, được cài và mở trên Samsung SM-A336E. Chính sách tải bản native mới được cập nhật; một bản vá Shorebird đã được xác nhận tải, áp dụng và tự khởi động lại ứng dụng trên thiết bị mà không cần cài lại APK.

Đây là triển khai nội bộ có kiểm soát. UAT đã được chủ dự án xác nhận; việc backend hiện có trên VPS không có nghĩa quy trình Continuous Deployment (CD) đã được tự động hóa. Hiện chưa triển khai CD tự động deploy lên VPS. Kịch bản phục hồi dữ liệu khi server gặp sự cố được chủ dự án thống nhất để xử lý sau. Bằng chứng triển khai và giới hạn được ghi trong [internal_free_deployment.md](./infrastructure/internal_free_deployment.md) và [production_readiness_plan.md](./release/production_readiness_plan.md).

## 6. Vận hành, bảo trì và hướng phát triển

Hệ thống đã có các thành phần nền cho vận hành: health check, monitor.service kiểm tra host/dịch vụ theo chu kỳ 5 phút và gửi cảnh báo về Discord khi một lần kiểm tra hoặc service liên quan thất bại, backup mã hóa lên Google Drive với lịch sử lưu tối đa hai ngày, cùng tài liệu runbook cho các tình huống đồng bộ hoặc message queue. Đây là giám sát theo chu kỳ, không phải cảnh báo tức thời cho mọi lần mất kết nối ngắn; chỉ báo khi monitor phát hiện sự cố ở lần kiểm tra tiếp theo. Vì monitor chạy trên VPS, nếu chính VPS mất hoàn toàn mạng/nguồn thì không thể đảm bảo service gửi được cảnh báo Discord; hiện chưa có monitor độc lập bên ngoài VPS. Quy trình release cũng lưu ý đến rollback ứng dụng và bảo vệ dữ liệu khi migration hoặc triển khai gặp lỗi.

Một số giới hạn vẫn cần được giữ rõ trong đánh giá hiện trạng. Backup được lưu trên Drive tối đa hai ngày, nhưng chưa triển khai kịch bản rollback/khôi phục dữ liệu khi server gặp sự cố; chủ dự án thống nhất có thể xử lý hạng mục này sau. Monitoring hiện có dựa trên monitor.service và cảnh báo Discord, chưa phải hệ thống giám sát độc lập bên ngoài VPS. Việc in hóa đơn vật lý và UAT đã được xác nhận. CD tự động deploy lên VPS chưa được triển khai; đây là trạng thái quy trình phát hành, không có nghĩa backend chưa được deploy. Các hạng mục chưa GO được ghi nhận là phần việc sẽ triển khai trong những đợt cập nhật tiếp theo, không phải phần đã hoàn tất trong phiên bản hiện tại.

Trong các đợt cập nhật tiếp theo, Panxii dự kiến lần lượt triển khai các hạng mục đang chờ GO như tự động hóa CD, hoàn thiện required merge checks/rollback vận hành, xử lý xung đột chỉnh sửa offline và kịch bản phục hồi dữ liệu server. Thứ tự và thời điểm cụ thể chưa được chốt trong báo cáo này. UAT đã xác nhận cho phạm vi hiện tại; điều đó không có nghĩa các hạng mục tương lai hoặc module mở rộng đã được nghiệm thu. Kho, chấm công, Bluetooth và thanh toán mở rộng vẫn thuộc phạm vi tiếp theo cho đến khi có thiết kế, triển khai và kiểm thử riêng.

### Checklist trạng thái GO / NO-GO

| Checklist | Trạng thái | Kết quả và giới hạn |
|---|---|---|
| [x] Backend và dịch vụ nền đã deploy | **GO cho vận hành nội bộ** | Backend .NET 10 đã deploy; health/readiness và Worker được kiểm tra. Không đồng nghĩa đã nghiệm thu toàn bộ pilot cửa hàng |
| [x] HTTPS, Nginx và bảo vệ port dịch vụ | **GO theo cấu hình/UAT đã xác nhận** | HTTPS đã cấu hình, DB/broker không mở public; UAT được chủ dự án xác nhận trong phạm vi đã thực hiện |
| [x] APK production và OTA | **GO** | APK 0.1.6+8 đã ký/phát hành nội bộ; bản vá OTA đã được xác nhận áp dụng và tự khởi động lại trên thiết bị |
| [x] In hóa đơn vật lý | **GO** | Chủ dự án xác nhận đã in thành công tại cửa hàng |
| [x] Synthetic soak và audit dữ liệu | **GO có giới hạn** | Run `20261104-100534` và final audit pass theo ngoại lệ được duyệt; không phải soak liên tục 24 giờ |
| [x] Backup định kỳ lên Drive | **GO cho việc tạo/lưu backup** | Có backup mã hóa, retention tối đa hai ngày; trạng thái này không thay cho diễn tập khôi phục server |
| [x] Monitor và cảnh báo Discord | **GO cho monitor hiện có** | Monitor kiểm tra mỗi 5 phút; cảnh báo được gửi khi phát hiện lỗi. Chưa có monitor độc lập ngoài VPS và không bảo đảm bắt mọi gián đoạn ngắn |
| [ ] Kịch bản phục hồi/rollback dữ liệu khi server lỗi | **Chưa GO — đợt cập nhật sau** | Chưa triển khai; chủ dự án đồng ý xử lý sau. Backup Drive hiện có không đồng nghĩa đã nghiệm thu khả năng khôi phục sự cố server |
| [x] UAT | **GO theo xác nhận của chủ dự án** | UAT đã được xác nhận; phạm vi là các luồng đã thực hiện trong đợt nghiệm thu |
| [ ] CI/CD — CD tự động deploy lên VPS | **Chưa GO — đợt cập nhật sau** | Backend đã chạy trên VPS, nhưng chưa có quy trình Continuous Deployment tự động phát hành lên VPS |
| [ ] Required merge checks và rollback vận hành | **Chưa GO — đợt cập nhật sau** | CI có kết quả pass trên commit cụ thể; cấu hình required merge checks và nghiệm thu đầy đủ quy trình rollback staging chưa được xác nhận hoàn tất |
| [ ] Hai thiết bị cùng sửa một sản phẩm, một thiết bị offline | **Chưa GO — đợt cập nhật sau** | Có optimistic concurrency và `ERR_VERSION_CONFLICT`; chưa có bài E2E đúng trình tự cùng chính sách merge thân thiện người dùng |
| [x] Trạng thái phát hành hiện tại | **GO theo phạm vi đã UAT** | UAT đã xác nhận; các hạng mục chưa GO được lên kế hoạch triển khai trong những đợt cập nhật tiếp theo, không được hiểu là đã có trong bản hiện tại |

## Kết luận

Sau sáu giai đoạn SDLC, Panxii đã hình thành một nền tảng POS offline-first có kiến trúc nhiều tenant, backend và mobile hoạt động cùng nhau, cơ chế đồng bộ có chống trùng, cùng bản triển khai nội bộ trên VPS và Android. Các bài kiểm thử backend và synthetic sync đã cung cấp bằng chứng đáng kể cho phần lõi.

Thành quả hiện tại đã có UAT xác nhận và phù hợp để vận hành theo phạm vi đã nghiệm thu. Backend đã deploy lên VPS, còn CD tự động và một số hạng mục khác chưa GO sẽ được triển khai ở những đợt cập nhật tiếp theo. Kịch bản phục hồi dữ liệu server được để lại xử lý sau theo thống nhất của chủ dự án; các hạng mục này không được mô tả như tính năng đã hoàn tất trong bản hiện tại.
