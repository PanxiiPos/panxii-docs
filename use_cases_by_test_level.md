# Use case Panxii theo từng level kiểm thử

Tài liệu này gom các use case đang có trong Panxii và mô tả cách kiểm chứng chúng ở từng tầng kiểm thử. Mục tiêu là giúp người đọc thấy rõ một chức năng được kiểm tra ở cấp nào, nội dung kiểm tra là gì và kết quả nào đã có bằng chứng.

## Cách đọc phạm vi và bằng chứng

- **Use case đã triển khai** mô tả khả năng hiện có; không đồng nghĩa mọi tầng kiểm thử của nó đều đã chạy hoặc có biên bản riêng.
- **Test case đặc tả** là kịch bản và kết quả mong đợi được ghi trong testing strategy hoặc role-based test cases. Nếu không có kết quả chạy cụ thể, không xem đó là test pass.
- **Kết quả tổng hợp** là số liệu suite đã chạy được ghi trong báo cáo dự án. Những số liệu tổng hợp không chứng minh từng test case trong bảng đều đã chạy.
- **UAT đã xác nhận** phản ánh xác nhận của chủ dự án cho phạm vi đã thực hiện; tài liệu hiện không có ma trận UAT chi tiết theo từng use case.
- Các use case chưa có UI mobile hoặc chưa hoàn tất E2E được ghi rõ giới hạn, không suy ra từ API hay tài liệu thiết kế.

## 1. Danh mục use case hiện có

| ID | Use case | Phạm vi hiện tại |
|---|---|---|
| UC-AUTH-01 | Đăng nhập và mở phiên thiết bị | Backend và luồng đăng nhập mobile cho TenantAdmin/Staff; SuperAdmin qua API nền tảng |
| UC-AUTH-02 | Refresh, logout và thu hồi phiên/device | Backend và mobile; device/session bị thu hồi không tiếp tục đồng bộ |
| UC-TENANT-01 | Onboard và quản lý tenant/TenantAdmin | API nền tảng cho SuperAdmin; tạo tenant và TenantAdmin đầu tiên theo transaction |
| UC-STAFF-01 | Quản lý tài khoản Staff | TenantAdmin quản lý Staff trong tenant hiện tại |
| UC-CAT-01 | Quản lý danh mục, sản phẩm và biến thể | Backend và mobile; gồm tạo/sửa, trạng thái active, tra cứu và bảo toàn lịch sử |
| UC-CAT-02 | Thay đổi giá theo phiên bản | Bulk price và optimistic concurrency; version cũ bị từ chối, không tự merge |
| UC-CUST-01 | Tìm kiếm và quản lý khách hàng | API/backend; mobile chưa có module quản lý khách hàng đầy đủ |
| UC-INV-01 | Tạo và lưu nháp hóa đơn | Mobile lưu draft và invoice đã chốt trong SQLite/SQLCipher |
| UC-INV-02 | Đồng bộ hóa đơn Sale | Mobile outbox → API/RabbitMQ → worker/MySQL → receipt/event/pull |
| UC-INV-03 | Tra cứu hóa đơn | API và mobile; Staff xem hóa đơn của mình, TenantAdmin xem trong tenant |
| UC-INV-04 | Đảo hoặc điều chỉnh hóa đơn | Backend; TenantAdmin tạo giao dịch reverse/correction, không sửa/xóa lịch sử gốc |
| UC-SYNC-01 | Retry, realtime và pull thay đổi | Idempotency, transactional outbox, SignalR theo tenant và pull theo sequence |
| UC-SET-01 | Cấu hình cửa hàng và effective settings | Backend/mobile theo quyền; cấu hình effective theo tenant override, global, application default |
| UC-SET-02 | Cập nhật feature flags | SuperAdmin quản lý global; TenantAdmin chỉ cập nhật tenant flags được phép |
| UC-PRINT-01 | In hóa đơn qua Windows Printer Bridge | Wi-Fi/LAN bridge đã triển khai; in vật lý được xác nhận thành công |
| UC-MOBILE-01 | Home, hồ sơ, trạng thái kết nối và cài đặt local | Mobile; ngôn ngữ/cỡ chữ là thiết lập local, tách khỏi cấu hình tenant |

Inventory, attendance, Bluetooth printing, payment processing và tự động merge xung đột không thuộc danh sách use case đã hoàn tất.

## 2. Level 1 — Unit test

Unit test kiểm tra quy tắc và kết quả của một thành phần riêng lẻ, không cần phụ thuộc MySQL, RabbitMQ, API thật hay máy in thật.

| Use case liên quan | Nội dung kiểm thử phù hợp | Tiêu chí chính |
|---|---|---|
| UC-INV-01, UC-INV-02 | Tính subtotal/total từ detail; kiểm tra quantity, unit price, discount, rounding và decimal | Sale có quantity/giá dương; thuế v1 bằng 0; tổng lệch detail trả đúng lỗi |
| UC-INV-04 | Tạo snapshot reverse với quantity/amount âm; tạo correction mới; ngăn reverse lặp | Không cập nhật hoặc xóa nội dung hóa đơn đã ghi nhận; reverse giữ liên kết hóa đơn gốc |
| UC-AUTH-01, UC-AUTH-02, UC-STAFF-01 | Quyền theo role; trạng thái user, tenant, device và session | Staff không được gọi thao tác dành cho TenantAdmin/SuperAdmin; revoked device bị chặn |
| UC-CAT-01, UC-CAT-02 | Tính giá bằng decimal, validate giá dương và version; áp dụng trạng thái product/variant | Version stale bị từ chối; giá hợp lệ theo policy; không cho Staff sửa giá |
| UC-CUST-01 | Validate dữ liệu customer và quyền thao tác | Staff chỉ đọc/tìm theo contract; thao tác ghi bị từ chối |
| UC-SET-01, UC-SET-02 | Hợp nhất setting theo thứ tự Tenant override → Global → Application default | Không có setting dùng default; kiểu value không đúng registry bị từ chối |
| UC-SYNC-01 | Idempotency key/payload hash, sequence, dedup event và xử lý version | Cùng operation/payload không tạo hiệu ứng lặp; cùng ID nhưng payload khác bị từ chối |

**Bằng chứng hiện có:** lượt `flutter test` ngày 06/11/2026 có 299 pass, 1 fail và 1 skip trên 301 test. Runner không xuất số pass riêng theo use case, nên không thể kết luận từng quy tắc trong bảng đã pass độc lập.

## 3. Level 2 — Integration test

Integration test kiểm tra sự phối hợp giữa các thành phần thật hoặc gần thật như API, MySQL/EF Core, RabbitMQ, worker, outbox, SignalR và database local.

| Use case liên quan | Thành phần và kịch bản | Tiêu chí chính |
|---|---|---|
| UC-AUTH-02, UC-TENANT-01, UC-STAFF-01 | API + MySQL: revoke, tenant lifecycle, lưu user và audit | Dữ liệu liên quan được commit cùng transaction; phiên cũ không còn quyền |
| UC-CAT-01, UC-CAT-02, UC-CUST-01, UC-SET-01 | API/worker + MySQL: tenant filter, uniqueness, version và rollback | Không đọc/ghi chéo tenant; version ngăn lost update; batch không commit một phần |
| UC-INV-02, UC-INV-04 | API + RabbitMQ + worker + MySQL: publisher confirm, idempotency, transaction invoice/change/outbox | `202` chỉ xác nhận queue nhận; entity, operation receipt, server change và outbox commit nhất quán |
| UC-SYNC-01 | Broker retry/DLQ, publisher restart, event dedup và pull sequence | Retry không tạo invoice trùng; lỗi publish sau DB commit không làm mất dữ liệu đã commit |
| UC-PRINT-01 | Adapter in giả lập với thành công/lỗi/timeout | Lỗi in không sửa hoặc rollback invoice; retry không tạo lệnh in ngoài ý muốn |
| UC-MOBILE-01, UC-INV-01 | SQLite/SQLCipher: local transaction, migration, force-stop/restart | Draft/outbox còn nguyên sau restart; database local không mất trạng thái chờ xử lý |

**Số liệu chạy trực tiếp ngày 06/11/2026:**

| Suite / lệnh | Pass | Fail | Skip | Tổng | Kết luận |
|---|---:|---:|---:|---:|---|
| Backend — `dotnet test Panxii.sln` | 230 | 0 | 25 | 255 | Toàn bộ test được thực thi đều pass; 25 test opt-in/đòi hỏi fixture không chạy trong lệnh mặc định |
| Mobile — `flutter test` | 299 | 1 | 1 | 301 | Chưa xanh toàn bộ; một widget test giao diện thất bại |

Gộp hai lượt chạy suite chuẩn: **529 pass, 1 fail, 26 skip trên 556 lượt test**. Đây là tổng lượt thực thi từ hai runner, không phải con số coverage hay số use case duy nhất.

Test mobile thất bại: `quantity and services are directly editable with keyboard on narrow large-text screen` trong `mobile/test/features/invoices/invoice_composer_test.dart`. Nút “Lưu và đồng bộ hóa đơn” nằm ngoài viewport 320×640, thao tác tap không tới được nút và test sau đó gặp null check. Không tính test này là pass.

Đây là tổng số theo runner, không phải phân bổ chính xác giữa Unit và Integration hoặc theo từng use case. Backend project gộp nhiều dạng test; Flutter `flutter test` chạy test/widget suite, không phải toàn bộ `integration_test`. Chưa có báo cáo runner phân bổ kết quả theo level/use case nên không suy diễn số pass cho từng hàng trong ma trận.

Các gate được ghi nhận riêng trong báo cáo nghiệm thu: backend resilience **23/23 pass, 0 skip**; MySQL provider compatibility **6/6 pass, 0 skip**; mobile development resilience **3 + 8 checkpoint**, đối chiếu **8 hóa đơn** không duplicate. Đây là test gate/checkpoint riêng, không cộng vào tổng `dotnet test`/`flutter test` phía trên.

## 4. Level 3 — API contract và role-based test

Level này xác minh request/response contract, HTTP status, error code, validation, quyền truy cập và tenant boundary. Nó kiểm tra API có tuân thủ hợp đồng, không chứng minh riêng trải nghiệm mobile hoàn chỉnh.

| Use case liên quan | Nội dung kiểm thử | Ví dụ tiêu chí |
|---|---|---|
| UC-AUTH-01, UC-AUTH-02 | Login, refresh, logout, token/session/device status | JWT thiếu/hết hạn hoặc device revoked bị từ chối; response không lộ credential |
| UC-TENANT-01, UC-STAFF-01 | Quyền platform/tenant, field được phép, optimistic concurrency | Role sai trả access denied; tenant B không thể bị thay đổi từ tenant A |
| UC-CAT-01, UC-CAT-02 | Schema product/category/variant, giá, barcode/SKU, expectedVersion | Field thừa bị từ chối; version cũ trả conflict; response và error envelope đúng contract |
| UC-CUST-01 | CRUD/search customer, giới hạn quyền Staff/TenantAdmin | Tenant scope đúng; hành động không được phép không tạo side effect |
| UC-INV-02, UC-INV-03, UC-INV-04 | Invoice sync/list/detail/reverse/correction | Idempotency key bắt buộc ở command; invoice immutable; quyền reverse chỉ TenantAdmin |
| UC-SET-01, UC-SET-02 | Effective settings và update flags | Global flag không bị tenant user chỉnh sửa; version/key/type được validate |
| UC-SYNC-01 | Pull cursor, status code và envelope | Pull trả sequence changes; invalid paging và lỗi dùng code ổn định |

**Bằng chứng hiện có:** [role_based_test_cases.md](testing/role_based_test_cases.md) chứa bộ case đặc tả SA/TA/ST/X cùng dữ liệu biên và kết quả mong đợi. Tài liệu này là test catalog/plan; không ghi execution report cho từng ID. Báo cáo dự án có số backend regression tổng hợp, nhưng không tách riêng số API contract tests đã pass.

## 5. Level 4 — End-to-end và resilience

E2E kiểm tra luồng đi qua nhiều lớp từ thao tác trên thiết bị đến trạng thái cuối ở server và thiết bị khác. Kịch bản trong bảng được mô tả trong testing strategy; trạng thái chạy thực tế phải đọc ở cột bằng chứng.

| Scenario | Use case | Luồng và tiêu chí |
|---|---|---|
| E2E-A: tạo invoice offline | UC-INV-01, UC-INV-02 | Tạo invoice khi offline → lưu local/outbox → force-stop/mở lại → bật mạng → server có đúng một invoice và local xác nhận Synced |
| E2E-B: timeout sau commit | UC-INV-02, UC-SYNC-01 | Mất response sau khi API nhận command → retry nguyên operation ID/payload → chỉ một invoice/change/outbox logic; event hoặc pull xác nhận kết quả |
| E2E-C: đứt mạng giữa batch | UC-INV-02, UC-SYNC-01 | Sync nhiều invoice theo batch → ngắt mạng → chỉ invoice chưa ack còn pending → nối lại và sync đủ, không duplicate |
| E2E-D: revoke thiết bị | UC-AUTH-02, UC-SYNC-01 | Revoke device có dữ liệu chờ → API từ chối với lỗi revoked → mobile dừng worker và chuyển về login theo policy |
| E2E-E: SignalR và pull | UC-INV-02, UC-SYNC-01 | Hai thiết bị cùng tenant nhận snapshot/sequence; tenant khác không nhận; khi realtime gián đoạn, pull bù thay đổi đã commit |
| E2E-F: hai thiết bị cùng sửa khi một thiết bị offline | UC-CAT-02 | Thiết bị offline gửi version cũ sau khi server đã có version mới → conflict hiển thị và không ghi đè server. Kịch bản E2E hoàn chỉnh và chính sách merge chưa được ghi nhận hoàn tất |
| E2E-G: reverse offline | UC-INV-04 | Cần quyết định cho phép hay cấm reverse offline; testing strategy ghi nhận đây là policy cần chốt, không coi là luồng đã nghiệm thu |

**Bằng chứng mobile development:** báo cáo ghi nhận gate mobile resilience gồm **3 + 8 checkpoint**, đồng bộ **8 hóa đơn** và không tạo duplicate ở server. Đây là kết quả trong môi trường development, không thay thế UAT cửa hàng hoặc chứng minh mọi scenario A–G đã chạy.

## 6. Level 5 — UAT và xác nhận nghiệp vụ

UAT kiểm tra xem các luồng được triển khai đáp ứng nhu cầu sử dụng thực tế của người dùng hay không; không thay thế unit/integration/contract tests.

| Use case được xác nhận ở mức dự án | Cách hiểu kết quả |
|---|---|
| Các luồng nằm trong phạm vi nghiệm thu hiện tại | Chủ dự án đã xác nhận UAT. Báo cáo không có biên bản chi tiết theo từng use case, thiết bị, người tham gia và từng bước kiểm thử |
| In hóa đơn | Đã xác nhận in thành công trên máy in vật lý |
| Hạng mục chưa GO và module mở rộng | Không được suy ra là đã UAT; được để cho các đợt cập nhật sau |

Vì không có danh sách UAT case-by-case được lưu trong tài liệu, trạng thái UAT ở đây chỉ nên trình bày là **đã xác nhận cho phạm vi đã thực hiện**, không gắn nhãn pass riêng cho tất cả use case trong danh mục.

## 7. Kiểm thử hiệu năng và soak

Soak/capacity là kiểm thử phi chức năng, bổ sung cho các level trên. Nó đo độ ổn định và độ trễ của các luồng read, pull và invoice sync; không thay thế kiểm tra đúng nghiệp vụ theo từng use case.

Ngưỡng được ghi nhận: **read p95 ≤ 500 ms**, **pull p95 ≤ 1.500 ms**, **invoice sync p95/max ≤ 1.500 ms**.

| Run | Thời lượng | Read | Pull | Invoice sync | Kết quả |
|---|---|---:|---:|---:|---|
| `20261103-135041` | Khoảng 40 phút 48 giây | Gặp `RemoteDisconnected`; không có percentile cuối | — | Cold invoice đầu tiên 827,05 ms | Fail do ngắt kết nối |
| `20261104-042043` | Khoảng 51 phút 58 giây | p95 1.344,62 ms | Không có p95 cho cửa sổ lỗi | 4 hóa đơn: 177,53 / 470,71 / 707,71 / 501,71 ms | Fail do read p95 vượt ngưỡng |
| `20261104-052131` | Khoảng 4 giờ 36 phút 56 giây | p95 cửa sổ lỗi 1.316,50 ms; p50 41,98 ms | p95 cửa sổ lỗi 11,21 ms | 20 hóa đơn mới; lớn nhất 530,60 ms | Fail do read p95 vượt ngưỡng |
| `20261104-100534` | 1 giờ 54 phút 26 giây | Max-window p95 94,05 ms | Max-window p95 46,52 ms | p95/max 300,57 ms | 109 cửa sổ; 8/8 hóa đơn xong; 0 reconnect/retry/transport failure; audit 139 bản ghi khớp |

Run cuối được chấp nhận để đóng gate synthetic soak/correctness theo ngoại lệ miễn thời lượng 24 giờ. Đây là **bounded soak pass**, không phải kết quả soak 24 giờ.

## 8. Ma trận tổng quan use case × level

| Use case | Unit | Integration | API contract / role | E2E / thiết bị | UAT |
|---|---|---|---|---|---|
| Authentication/session/device | Quy tắc token, role, revoke | Session/device state và DB | Login/refresh/logout, 401/403 | Revoke-device scenario được đặc tả | UAT xác nhận chung, không có case riêng được công bố |
| Tenant/TenantAdmin/Staff | Quy tắc quyền và version | Tenant transaction, uniqueness, revoke | SA/TA role cases | Không có kết quả E2E riêng trong báo cáo | Không có mapping case riêng |
| Catalog/category/pricing | Giá, validation, version | MySQL, tenant scope, transaction | TA/ST cases | Mobile catalog offline; conflict hai thiết bị chưa hoàn tất E2E | Không có mapping case riêng |
| Customer | Validation và quyền | Lưu/search/customer lifecycle | API role cases | Mobile customer flow chưa có đầy đủ | Không có mapping case riêng |
| Invoice create/sync | Total, snapshot, idempotency | DB + broker + worker + outbox | Invoice schema, status, role/error | 8 invoice trong mobile resilience; offline scenarios đặc tả | UAT xác nhận chung |
| Invoice reverse/correction | Dấu, immutability, one-reverse rule | Concurrent reverse, atomic correction | TenantAdmin-only contract | Reverse offline còn policy cần chốt | Không có mapping case riêng |
| Sync/realtime/pull | Dedup/version/sequence | Retry/DLQ/outbox/SignalR | Pull cursor/error contract | Checkpoint mobile và soak run | UAT xác nhận chung |
| Settings/feature flags | Effective value/type | DB/version/audit | Role, scope, schema | Không có kết quả E2E riêng được nêu | Không có mapping case riêng |
| Printing | Lỗi không làm đổi invoice | Adapter printer giả lập | Quyền và flag theo contract | In vật lý đã xác nhận; Bluetooth chưa triển khai | Xác nhận in vật lý |
| Home/profile/local settings | State/render/validation | Cache/API/profile persistence | Profile/settings contract | Một phần thuộc mobile development gate | Không có mapping case riêng |

Ma trận này là chỉ mục phạm vi và bằng chứng hiện có, không phải báo cáo coverage phần trăm. Tổng pass/fail/skip của suite được ghi ở mục 3; chưa có báo cáo theo test ID để phân bổ chính xác cho từng use case. Các cụm “đặc tả” hoặc “không có mapping” không đồng nghĩa đã chạy test.

## Tài liệu tham chiếu

- [Use case catalog và domain model](architecture/domain_use_cases.md)
- [Các module đã triển khai và logic xử lý](implemented_modules.md)
- [Testing strategy và test pyramid](testing/testing_strategy.md)
- [Role-based functional test cases](testing/role_based_test_cases.md)
- [Risk-based testing specification](testing/risk_based_testing_spec.md)
- [API contract](api/api_contract.md)
- [Báo cáo kết quả dự án và số liệu kiểm thử](result.md)
