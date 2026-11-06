# Các module đã triển khai và logic xử lý

Tài liệu này mô tả các module có trong Panxii và cách dữ liệu đi qua ứng dụng mobile, API, worker và database. Đây là phần giải thích chức năng, không thay thế API contract, business rules hay hướng dẫn vận hành.

## Cách đọc trạng thái

- **Đã triển khai:** module có xử lý ở backend và/hoặc mobile như ghi rõ trong cột phạm vi.
- **Backend có, mobile giới hạn:** API và nghiệp vụ server tồn tại nhưng chưa có đầy đủ màn hình/luồng thao tác trên ứng dụng.
- **Đã triển khai có điều kiện:** có thành phần hoặc contract, nhưng chỉ nên bật khi backend, client và cấu hình liên quan cùng hỗ trợ.
- **Chưa triển khai:** có thể được nhắc trong tài liệu thiết kế nhưng chưa được xem là chức năng sử dụng được.

Các quy tắc bảo mật và tính đúng dữ liệu được thực thi ở server. Cache, trạng thái UI hoặc feature flag trên thiết bị không thay thế authorization hay validation của API.

## Bản đồ module

| Module | Trạng thái | Vai trò sử dụng chính |
|---|---|---|
| Đăng nhập và phiên thiết bị | Đã triển khai backend và mobile | SuperAdmin qua API nền tảng; TenantAdmin, Staff trên mobile |
| Quản trị tenant và TenantAdmin | Đã triển khai backend | SuperAdmin |
| Quản lý Staff | Đã triển khai backend và luồng quản trị cửa hàng | TenantAdmin |
| Sản phẩm, biến thể và danh mục | Đã triển khai backend và mobile | TenantAdmin quản lý; Staff tra cứu |
| Khách hàng | Backend có; mobile chưa có module quản lý đầy đủ | TenantAdmin qua API; Staff API theo contract |
| Hóa đơn | Đã triển khai backend và mobile | Staff lập hóa đơn; TenantAdmin quản lý và đảo/điều chỉnh |
| Offline sync và realtime | Đã triển khai | Worker thiết bị, API, worker server, các thiết bị cùng tenant |
| Cấu hình cửa hàng và feature flags | Đã triển khai backend và mobile, theo quyền | SuperAdmin cấu hình global; TenantAdmin cấu hình tenant |
| In hóa đơn qua Wi-Fi | Đã triển khai; in vật lý đã được xác nhận | Staff/TenantAdmin theo quyền và cờ tenant |
| Home, hồ sơ, kết nối và đa ngôn ngữ | Đã triển khai mobile/backend | TenantAdmin, Staff |

## 1. Đăng nhập, xác thực và phiên thiết bị

**Phạm vi:** Backend xác thực tenant user và SuperAdmin; mobile hiện hỗ trợ luồng cửa hàng cho TenantAdmin/Staff.

### Logic xử lý

1. Người dùng gửi username/password đến `POST /auth/login`. Client không tự chọn tenant; server phân giải tài khoản, tenant, role và device.
2. Server cấp access token JWT và refresh token opaque. Access token chứa các claim cần cho authorization; refresh token chỉ được lưu dạng hash ở server.
3. Mobile lưu access token, refresh token và thời điểm hết hạn trong secure storage, không lưu mật khẩu. Trước request API hoặc kết nối SignalR, app tự refresh khi access token sắp hết hạn.
4. Khi API trả 401, client refresh rồi gửi lại request tối đa một lần với cùng payload và idempotency key. Nhiều request đồng thời dùng chung một lần refresh.
5. Logout gọi `POST /auth/logout`, xác nhận revoke thành công rồi xóa token local. Lỗi mạng giữ phiên để người dùng thử lại.
6. Nếu tài khoản, tenant hoặc device bị khóa/revoke, server từ chối request. Client khóa request/realtime, xóa thông tin phiên và điều hướng về Login theo luồng bắt buộc.

### Quyền và giới hạn

- JWT TenantAdmin/Staff có tenant context; server lấy tenant từ token thay vì tin `tenantId` trong request.
- SuperAdmin là danh tính platform, không phải user thuộc một tenant cửa hàng.
- Revoke device/session có hiệu lực ở server; cache local không thể cấp lại quyền.
- App hiện từ chối lưu các role ngoài Staff/TenantAdmin vào luồng mobile cửa hàng.

## 2. Quản trị nền tảng và vòng đời tenant

**Phạm vi:** API SuperAdmin quản lý tenant, các TenantAdmin của tenant và global feature flags.

### Tạo tenant và onboarding

Tạo tenant cùng TenantAdmin đầu tiên là một transaction duy nhất. Nếu tạo tài khoản quản trị thất bại, tenant cũng không được lưu dở dang. Mật khẩu ban đầu chỉ dùng để tạo tài khoản; API không trả hoặc ghi log credential.

### Cập nhật và vòng đời

- SuperAdmin có thể tìm, xem và cập nhật metadata tenant với `expectedVersion`.
- Suspend tenant đổi trạng thái và revoke device/session liên quan; token/session cũ không được tiếp tục dùng.
- Reactivate tenant không phục hồi session đã revoke; user cần đăng nhập lại.
- Xóa tenant là soft-delete: đóng tenant, revoke session và giữ nguyên dữ liệu nghiệp vụ/audit.
- Quản lý tài khoản TenantAdmin theo tenant đích; không cho endpoint này thay đổi role.
- Cập nhật version cũ bị từ chối bằng `ERR_VERSION_CONFLICT`.

Các thao tác platform là đường quyền riêng, có audit; không tạo một tenant giả để cấp quyền SuperAdmin.

## 3. Quản lý tài khoản Staff

**Phạm vi:** TenantAdmin quản lý Staff thuộc tenant hiện tại.

- Danh sách hỗ trợ tìm kiếm theo username/tên hiển thị và phân trang; response không chứa mật khẩu hoặc token.
- Tạo Staff bằng thông tin tài khoản và mật khẩu khởi tạo. Username duy nhất trong tenant.
- Cập nhật hồ sơ Staff dùng `expectedVersion`; request không được tự đổi tenant, role hoặc credential.
- Deactivate Staff giữ tham chiếu lịch sử và revoke các device/session ngay lập tức.
- Chỉ hard-delete khi tài khoản chưa có lịch sử hoặc dependency cần giữ; nếu không, server yêu cầu deactivate.
- Tất cả thao tác xác định tenant từ JWT và kiểm tra quyền ở backend.

Mobile staff management dành cho TenantAdmin; Staff không tự quản lý tài khoản hoặc quyền của người khác.

## 4. Danh mục, sản phẩm và biến thể

**Phạm vi:** Backend và mobile đã có luồng quản lý catalog.

### Mô hình dữ liệu

- `Product` là nhóm hàng gốc; `ProductVariant` là đơn vị bán được.
- SKU, barcode, tên biến thể, đơn vị, giá và thuộc tính đặc điểm thuộc variant. Product giữ thuộc tính dùng chung và quan hệ danh mục.
- Product mới tạo cùng default variant trong một lần xử lý. Default variant dùng chung ID với product để tương thích các client cũ.
- Đặc điểm mở rộng lưu theo `ExtraData` JSON versioned; client giữ lại dữ liệu/field chưa biết thay vì tự loại bỏ.

### Danh mục và tra cứu

- Mobile TenantAdmin quản lý danh mục; danh sách catalog được lọc theo tab danh mục active đang chọn, không trộn dữ liệu giữa danh mục.
- Tạo sản phẩm từ mobile yêu cầu danh mục active. Mã hàng bỏ trống được sinh một lần trước khi enqueue và được giữ nguyên qua retry.
- Staff chỉ xem sản phẩm/variant đang bán. TenantAdmin có thể xem cả inactive để quản lý.
- Search catalog tìm tên nhóm, tên variant, SKU, barcode và thuộc tính được lập chỉ mục; kết quả có cursor pagination.
- Local cache phân biệt product với variant và được phân vùng tenant/user.

### Giá và thay đổi catalog

- Giá bán được tính từ giá niêm yết nhân hệ số quy đổi theo rule của sản phẩm/variant; API validation dùng decimal, không dùng floating point cho phép tính tiền.
- Category có tùy chọn làm tròn theo nghìn nếu được bật theo policy; không suy luận từ tên sản phẩm.
- Bulk price update có thể chọn product, variant hoặc cả hai; product/default variant không bị điều chỉnh lặp trong cùng giao dịch.
- Sửa giá, sửa thông tin, pause/resume và xóa cứng có kiểm tra version/quan hệ/lịch sử. Product hoặc variant có lịch sử hóa đơn cần giữ lại và ngừng bán thay vì xóa.
- Thay đổi catalog trên mobile được ghi vào journal local; worker gửi command cùng operation ID. Server kiểm tra role, tenant, version và rule lần nữa trước khi commit.

### Xung đột nhiều thiết bị

Mỗi mutation mang `expectedVersion`. Nếu một thiết bị offline sửa bản version 4, trong khi thiết bị khác đã cập nhật server lên version 5, lệnh offline cũ bị từ chối với `ERR_VERSION_CONFLICT`. Client giữ trạng thái lỗi để người dùng xử lý; không tự ghi đè version server hoặc tự rebase payload. Chính sách merge tự động chưa được triển khai.

## 5. Khách hàng

**Phạm vi:** Backend có API khách hàng; mobile chưa có module quản lý khách hàng đầy đủ. Luồng hóa đơn mobile hiện dùng khách vãng lai (`customerId: null`).

- API hỗ trợ đọc/tìm khách hàng trong tenant; TenantAdmin có thể quản lý vòng đời và hồ sơ khách.
- Tạo customer dùng command idempotent qua RabbitMQ, nên phản hồi `202 Queued` chỉ xác nhận tiếp nhận chứ chưa xác nhận đã tạo xong.
- Cập nhật dùng optimistic concurrency (`expectedVersion`); version cũ trả lỗi conflict.
- Customer đã được tham chiếu trong hóa đơn không bị hard-delete; dùng deactivate để giữ lịch sử.
- Staff chỉ có quyền theo contract backend; không được sửa/xóa customer. Feature yêu cầu số điện thoại được kiểm tra lại tại server khi sync hóa đơn.
- Chưa có màn hình mobile danh sách/chi tiết/form customer hoặc luồng chọn khách trong invoice được xem là hoàn chỉnh.

## 6. Lập hóa đơn và lịch sử giao dịch

**Phạm vi:** Mobile hỗ trợ draft, tra cứu, detail và sync Sale; backend xử lý ghi sổ, reverse và correction.

### Draft và xác nhận Sale

1. Người dùng chọn các variant đang bán, nhập quantity và các thành phần tổng tiền.
2. Draft chỉ lưu SQLite/SQLCipher; mở composer trống không tự tạo draft.
3. Khi chốt, app tạo invoice ID và operation ID ổn định, đồng thời ghi invoice, detail snapshots và local outbox trong cùng transaction.
4. Mỗi detail lưu ID product/variant và snapshot SKU, tên, đơn vị, giá cùng thuộc tính tại thời điểm bán. Thay đổi catalog sau này không viết lại lịch sử hóa đơn.
5. Sale quantity và unit price phải dương. Thuế trong v1 bằng 0. Server kiểm tra lại catalog, snapshot, tổng detail và total trước khi ghi.
6. Invoice content đã commit trên server là immutable; API không hỗ trợ sửa/xóa nội dung Sale đã đồng bộ.

### Trạng thái

| Trạng thái | Ý nghĩa |
|---|---|
| Draft | Chỉ có trên thiết bị, có thể sửa/xóa local |
| Pending / Sending | Đã chốt local hoặc đang được client gửi |
| Queued | API đã xác nhận RabbitMQ nhận lệnh; server có thể chưa commit |
| Synced | Đã xác nhận server commit qua receipt, SignalR hoặc pull |
| Failed | Server từ chối hoặc cần người dùng xử lý; giữ error code và dữ liệu local |

Trạng thái nghiệp vụ server `Active`/`Reversed` là khái niệm riêng, không dùng thay `SyncStatus`.

### Danh sách và tra cứu

- Staff chỉ đọc hóa đơn do chính mình tạo; TenantAdmin đọc hóa đơn trong tenant.
- Danh sách hỗ trợ tìm theo số hóa đơn và phân trang; chi tiết render từ snapshot đã lưu.
- Invoice khách vãng lai giữ thông tin customer trống; không tạo customer ngầm.
- QR tra cứu số hóa đơn hoặc VietQR trên bản in không phải bằng chứng đã thanh toán. Hệ thống hiện không có payment status.

### Reverse và correction

- TenantAdmin mới được reverse/correct; Staff không có quyền này.
- Reverse tạo invoice mới có giá trị/quantity âm và liên kết `OriginalInvoiceId`; cập nhật trạng thái nghiệp vụ theo cặp đảo trong cùng transaction, không sửa/xóa detail gốc.
- Correction thực hiện reverse hóa đơn cũ và tạo hóa đơn mới từ detail đã điều chỉnh, đồng thời ghi audit/change/outbox atomically.
- Cả hai thao tác yêu cầu idempotency key. Retry cùng key/payload trả kết quả đã xử lý; một hóa đơn gốc không được reverse lặp.

### Phí dịch vụ

Receipt template có thể mô tả service blocks. Invoice payload v2 hỗ trợ snapshot phí dịch vụ theo invoice để giữ nhãn/số tiền lịch sử; v1 không phí giữ tương thích. Chỉ bật luồng có phí khi phiên bản mobile, API và worker trên môi trường đích đều hỗ trợ payload v2.

## 7. Offline sync, command queue và realtime

Đây là luồng chung cho invoice và các catalog/customer command đã được đưa qua hàng đợi.

### Luồng từ thiết bị lên server

1. Mobile commit dữ liệu local và outbox trước khi gọi mạng.
2. Outbox giữ `ClientOperationId`, payload đã đóng băng, payload hash và trạng thái retry.
3. Client gửi command với `Idempotency-Key`. API chỉ trả `202 Queued` sau khi RabbitMQ publisher confirm.
4. Worker xác thực lại tenant, actor, role, payload/version và nghiệp vụ.
5. Worker commit entity, operation receipt, `server_changes` sequence và `outbox_events` trong cùng transaction MySQL.
6. Worker outbox publisher phát snapshot sau commit. Event có stable message ID và server sequence.
7. Client chỉ đánh dấu Synced sau receipt/event/pull chứng minh lệnh hoàn tất; không coi online hoặc HTTP 202 là thành công cuối.

### Retry và lỗi

- Lỗi mạng/timeout giữ nguyên operation ID và payload để retry; không tạo UUID mới cho một lần retry thông thường.
- Cùng operation ID nhưng payload khác bị từ chối bằng `ERR_SYNC_PAYLOAD_MISMATCH`.
- Lỗi nghiệp vụ như catalog/version conflict không retry vô hạn; giữ dữ liệu và hiển thị lỗi để người dùng xử lý.
- Command tạm lỗi có retry queue; thông điệp không hợp lệ hoặc vượt retry có thể vào DLQ để vận hành kiểm tra.
- Nếu publish realtime lỗi sau khi DB commit, outbox còn pending để phát lại; dữ liệu đã commit không bị rollback vì SignalR/broker.

### Realtime và pull

- SignalR chỉ gửi đến group lấy từ tenant claim; không dùng tenant ID client tự khai để chọn group.
- Snapshot realtime giúp thiết bị cùng tenant cập nhật nhanh, nhưng không phải nguồn dữ liệu duy nhất.
- Sau reconnect hoặc bỏ lỡ event, client gọi `GET /sync/pull?afterSequence=...`.
- Client upsert theo entity version/message ID rồi chỉ tăng local sequence sau khi ghi local thành công. Event cũ/trùng không được làm lùi version hay sequence.

## 8. Cấu hình cửa hàng và feature flags

### Cấu hình tenant

TenantAdmin quản lý các nhóm cài đặt như StoreProfile, ReceiptTemplate, ProductDisplay và các tenant flags được cho phép. Staff chỉ đọc cấu hình effective.

- API hợp nhất cấu hình theo thứ tự `Tenant override > Global > Application default`.
- Mỗi key có version riêng. PUT gửi `expectedVersion` của đúng key; version cũ trả conflict thay vì ghi đè.
- Mobile cache cấu hình theo tenant trong SQLCipher. Khi lưu, app ghi draft local trước khi PUT; lỗi mạng giữ draft để thử lại. Đây là local draft, không phải background outbox.
- Server mới là nơi quyết định quyền và validate schema; cấu hình local không cấp quyền.

### Feature flags

- SuperAdmin quản lý global flags; TenantAdmin chỉ quản lý tenant flags được cấp phép.
- Server enforce flag ở API/worker; chỉ ẩn nút trên UI là không đủ.
- `EnableWifiPrint` mặc định tắt và được dùng để kiểm soát in qua PC. `EnableBluetoothPrint` chỉ là tùy chọn cấu hình; Bluetooth transport chưa được triển khai.
- `RequireCustomerPhone` và các global flags không được mobile editor tự biến thành tenant override.

## 9. In hóa đơn qua Windows Printer Bridge

**Trạng thái:** luồng Wi-Fi bridge đã triển khai; chủ dự án xác nhận đã in thành công trên máy in vật lý.

1. Staff/TenantAdmin mở hóa đơn đã ghi nhận, xem preview và chọn in khi `EnableWifiPrint` được server xác nhận.
2. Mobile dựng PNG A4 từ cùng renderer dùng cho preview, dựa trên invoice snapshots và cấu hình cửa hàng/mẫu in.
3. Ứng dụng ghép nối với Windows PC trong LAN qua HTTPS, dùng certificate pin/token. Bridge gửi tài liệu cho Windows print queue/driver; không cần máy in có Wi-Fi.
4. SQLite job dedup và UUID ổn định ngăn retry job tạo lệnh in trùng. Trạng thái “submitted” chỉ xác nhận queue Windows đã nhận; trạng thái không chắc chắn không tự động in lại.
5. Lỗi máy in không rollback hoặc sửa hóa đơn. QR chuyển khoản nếu có chỉ chứa thông tin thanh toán tĩnh, không xác nhận giao dịch ngân hàng.

Bridge chạy trên PC Windows, không chạy trong Docker; firewall chỉ mở cho mạng Private/LocalSubnet. Bluetooth/ESC-POS và PDF export chưa được triển khai.

## 10. Home, hồ sơ, kết nối và trải nghiệm mobile

- Home tổng hợp lời chào từ hồ sơ, metadata tenant và effective settings; snapshot cache phân vùng tenant/user cho phép hiển thị dữ liệu đã tải trước khi mất mạng.
- Cache chỉ dùng cho lỗi kết nối phù hợp; lỗi 401/403, session revoked, identity/contract sai không được che bằng dữ liệu cũ.
- Hồ sơ cá nhân cho phép cập nhật trường được cấp quyền với optimistic concurrency; username, role và tenant không được sửa qua hồ sơ.
- Connection status và trạng thái sync là riêng biệt: có Internet không đồng nghĩa dữ liệu đã đồng bộ.
- Lỗi API dùng error code để dịch bằng `easy_localization` (vi/en); response có correlation ID để truy vết.
- Cỡ chữ và ngôn ngữ lưu trên thiết bị; cập nhật giao diện không thay đổi tenant settings hay tạo sync operation.

## 11. Những module không nên hiểu là đã triển khai

- Quản lý customer đầy đủ trên mobile và chọn customer trong hóa đơn.
- Inventory/kho và attendance/chấm công.
- Bluetooth printer transport.
- Payment processing, xác nhận thanh toán ngân hàng hoặc trạng thái Paid/Unpaid.
- Tự động merge mọi thay đổi khi hai thiết bị offline/online cùng sửa một record.
- CD tự động deploy backend lên VPS.
- Phục hồi/rollback dữ liệu server khi có sự cố; backup có sẵn không đồng nghĩa đã nghiệm thu restore production.

## Tài liệu tham chiếu

- [Tổng quan sản phẩm](project_overview.md)
- [Quy tắc nghiệp vụ](business_rule.md)
- [Domain model và use cases](architecture/domain_use_cases.md)
- [API contract](api/api_contract.md)
- [Mobile screen flow và API mapping](frontend/mobile_screen_flow_api_mapping.md)
- [Đồng bộ RabbitMQ và SignalR](sync/data_sync_flow_rabbitmq.md)
- [Security, RBAC và feature flags](security/security_rbac_feature_flags.md)
- [Database design](database/database_design.md)
- [Tổng kết dự án và trạng thái nghiệm thu](result.md)
