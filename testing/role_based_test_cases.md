# ROLE-BASED FUNCTIONAL TEST CASES

**Mục tiêu:** Chuẩn bị test case cho các chức năng SuperAdmin, TenantAdmin và Staff trước khi triển khai; bao phủ quyền truy cập, dữ liệu biên, giá trị dương/âm, dữ liệu sai, field thiếu/thừa và tenant isolation.

**Nguồn quy tắc:** `business_rule.md` là nguồn nghiệp vụ ưu tiên theo `README.md`. Dùng `api_contract.md`, `openapi.yaml`, `error_codes.md` và `database_design.md` để lấy request shape, giới hạn kỹ thuật và error code đã công bố.

## 1. Quy ước chạy test

- Mỗi test tạo dữ liệu cô lập; không dùng PII thật. Ghi lại role, tenant, request, HTTP status, `error.code`, thay đổi DB, audit và session/device bị revoke.
- Request thành công phải trả envelope chuẩn. Request lỗi phải dùng error code, không phụ thuộc nội dung message.
- Với quyền bị từ chối: mong đợi `403 ERR_ACCESS_DENIED`, dữ liệu không đổi và không tạo side effect. Resource ngoài tenant trả `404` theo contract, không làm lộ dữ liệu.
- Test các field bắt buộc bằng ba biến thể: bỏ field, `null`, chuỗi rỗng. Field nullable phải phân biệt bỏ qua field với gửi `null` để xóa giá trị.
- Với field có min/max: thử đúng min, đúng max, min-1 và max+1. Với số: thử 0, giá trị dương nhỏ nhất, giá trị âm nhỏ nhất, giới hạn biểu diễn và vượt giới hạn.
- Từ chối mọi JSON field không khai báo trong request schema, kể cả field lồng nhau, bằng `400 ERR_UNKNOWN_FIELD`; `fieldErrors` nêu đường dẫn field. Field cấm không được mass-assign.
- Dùng error code chuyên biệt cho từng nhóm validation; không chỉ assert HTTP status hoặc dùng message text.
- Nếu contract chưa định nghĩa giới hạn nghiệp vụ, ghi kết quả là **TBD**; không tự xem giới hạn kỹ thuật của database là chính sách kinh doanh.

### 1.1. Dữ liệu biên dùng chung

| Dữ liệu | Giá trị kiểm thử | Mong đợi |
|---|---|---|
| `page`, `pageSize` | `page=1`, `pageSize=1` và `200` | Hợp lệ; paging đúng |
| `page`, `pageSize` | `0`, `-1`, `201`, chữ hoặc số vượt integer | `400 ERR_INVALID_PAGING` |
| `search` | rỗng, 1 ký tự, đúng 120, 121 ký tự | 120 ký tự trở xuống hợp lệ; 121 ký tự trả `400 ERR_INVALID_FIELD` |
| Giá `DECIMAL(19,4)` | `0`, `0.0001`, số âm, `999999999999999.9999`, overflow, `1.23456` | Giá phải `> 0`; zero/âm trả `422 ERR_PRICE_MUST_BE_POSITIVE`; overflow/precision >4 trả `400 ERR_INVALID_DECIMAL` |
| Số lượng `DECIMAL(19,4)` | `0`, `0.0001`, số âm, overflow, `1.23456` | Sale chỉ quantity dương; 0/âm trả `422 ERR_INVALID_QUANTITY`; Reverse âm chỉ do server tạo; precision/range sai trả `ERR_INVALID_DECIMAL` |
| Thuế trong v1 | `taxRate=0`, `taxAmount=0`; thử gửi giá trị khác 0 | Chưa kiểm thử/tính nghiệp vụ thuế; v1 giữ các field tương thích ở 0. Thuế là phạm vi mở rộng sau |
| UUID / enum / email / timezone | UUID sai; enum lạ; email sai; timezone sai | `ERR_INVALID_UUID`, `ERR_INVALID_ENUM`, `ERR_INVALID_EMAIL`, `ERR_INVALID_TIME_ZONE` tương ứng |
| Required/unknown/type | Bỏ/null/rỗng required; field thừa; sai kiểu | `ERR_REQUIRED_FIELD`, `ERR_UNKNOWN_FIELD`, `ERR_INVALID_FIELD` tương ứng |
| `expectedVersion` | hiện tại, cũ, thiếu, `0`, số âm, chuỗi | Current hợp lệ; stale trả `409 ERR_VERSION_CONFLICT`; thiếu trả `ERR_REQUIRED_FIELD`; sai kiểu/range trả `ERR_INVALID_FIELD` |

## 2. Test fixture

- Tenant A: SuperAdmin global, TenantAdmin A, Staff A; có product active/inactive, customer chưa/đã gắn invoice, invoice Sale đã sync và invoice đã reverse.
- Tenant B: TenantAdmin B, Staff B và dữ liệu cùng loại với ID/SKU/code khác; dùng để thử cách ly tenant.
- Có tài khoản TenantAdmin/Staff chưa có lịch sử và đã có lịch sử/device/session.
- Có ít nhất hai global feature flags đã đăng ký, một key không tồn tại, và setting tenant có version hiện tại.
- Có thiết bị in local giả lập thành công, timeout và Bluetooth bị tắt.

## 3. SuperAdmin

| ID | Chức năng / dữ liệu | Kết quả mong đợi |
|---|---|---|
| SA-001 | Liệt kê tenant, mặc định và `includeDeleted=true`; kiểm tra page/pageSize/search ở biên | Chỉ SuperAdmin truy cập; mặc định loại tenant đã xóa mềm, includeDeleted trả cả tenant đã xóa; paging đúng |
| SA-002 | Tạo tenant với code/name/timeZone/currencyCode và TenantAdmin hợp lệ | `201`; tenant Active version 1; TenantAdmin cùng transaction; response không có password/hash/token |
| SA-003 | Tạo tenant với code 1/32 ký tự, name 1/160, timezone 1/64, currency đúng 3 chữ cái | Biên hợp lệ được chấp nhận; ngoài độ dài trả `ERR_INVALID_FIELD`; currency sai pattern `ERR_INVALID_FIELD`; timezone sai `ERR_INVALID_TIME_ZONE` |
| SA-004 | Tạo tenant thiếu/null/rỗng từng field bắt buộc ở root và `tenantAdmin`; gửi field thừa `tenantId`, `role`, `status`, credential khác | Required trả `ERR_REQUIRED_FIELD`; field thừa trả `ERR_UNKNOWN_FIELD`; không tạo tenant/account và rollback toàn transaction |
| SA-005 | Code trùng toàn hệ thống hoặc username trùng trong tenant mới | `409 ERR_DUPLICATE_TENANT_CODE` hoặc `ERR_DUPLICATE_USERNAME`; không lưu một phần |
| SA-006 | Password ban đầu dài 11, 12, 128, 129 ký tự | 12 và 128 hợp lệ; ngoài range trả `400 ERR_INVALID_FIELD`; password không trả về/ghi log |
| SA-007 | Sửa tenant: boundary các field, expectedVersion đúng/stale/thiếu/0/âm; gửi `status`/`isDeleted` ngoài request | Giá trị hợp lệ được cập nhật và tăng version; stale `409 ERR_VERSION_CONFLICT`; field thừa trả `400 ERR_UNKNOWN_FIELD` |
| SA-008 | Suspend tenant đang có session; gọi API bằng access/refresh token cũ | Tenant Suspended; session/device bị revoke; token cũ bị chặn với `401 ERR_TENANT_INACTIVE` |
| SA-009 | Reactivate tenant Suspended và thử tenant Closed/soft-deleted | Suspended chuyển Active; session cũ không tự sống lại; tenant đã soft-delete không reactivate được |
| SA-010 | Xóa tenant có dữ liệu; đọc lại có/không `includeDeleted` | Soft-delete, status Closed, audit được ghi, toàn bộ session bị revoke; dữ liệu nghiệp vụ còn nguyên |
| SA-011 | Xem danh sách/chi tiết TenantAdmin trong đúng tenant, kể cả account inactive | Trả đúng tenant; không lộ password/hash/PIN/token; tenant khác trả 404 |
| SA-012 | Tạo TenantAdmin với username/displayName/password/email/phone hợp lệ ở biên | `201`; username unique trong tenant; password chỉ hash, không response/log |
| SA-013 | Tạo TenantAdmin với username/displayName/password thiếu hoặc ngoài min/max; email sai; gửi `tenantId`/`role` | Thiếu trả `ERR_REQUIRED_FIELD`; password range sai `ERR_INVALID_FIELD`; email sai `ERR_INVALID_EMAIL`; field thừa `ERR_UNKNOWN_FIELD`; không tạo account |
| SA-014 | Sửa TenantAdmin với expectedVersion hiện tại/stale/thiếu; gửi status, tenantId, role hoặc credential | Cập nhật field được phép; stale `409 ERR_VERSION_CONFLICT`; field thừa `400 ERR_UNKNOWN_FIELD`; không đổi field cấm |
| SA-015 | Vô hiệu hóa TenantAdmin đang đăng nhập trên nhiều device | Status Inactive, revoke ngay mọi session/device, ghi audit; token hiện hành trả `401 ERR_DEVICE_REVOKED` |
| SA-016 | Xóa TenantAdmin chưa có dependency và đã có history/device/session | Chưa dùng: hard-delete theo contract; đã dùng: `409 ERR_TENANT_ADMIN_DELETE_REQUIRES_DEACTIVATION`, history/audit giữ nguyên |
| SA-017 | Đọc/cập nhật global flag bằng SuperAdmin; đổi `true -> false -> true` | Giá trị đọc lại đúng, version tăng và audit được ghi |
| SA-018 | Global flag: key không đăng ký, value sai type, expectedVersion thiếu/stale/âm; gửi field thừa | Key lạ `404 ERR_FEATURE_FLAG_NOT_FOUND`; type/range sai `ERR_INVALID_FIELD`; thiếu `ERR_REQUIRED_FIELD`; stale `409 ERR_VERSION_CONFLICT`; field thừa `ERR_UNKNOWN_FIELD` |
| SA-019 | TenantAdmin/Staff gọi API platform tenant/admin/global flag | `403 ERR_ACCESS_DENIED`; không có thay đổi hoặc lộ dữ liệu platform |

## 4. TenantAdmin

| ID | Chức năng / dữ liệu | Kết quả mong đợi |
|---|---|---|
| TA-001 | Xem/tìm kiếm product trong tenant; mặc định chỉ active, `includeInactive=true` xem inactive | Trả đúng tenant; Staff/TenantAdmin không thấy dữ liệu tenant khác; inactive không được dùng tạo Sale invoice |
| TA-002 | Tạo product với SKU/name/unit ở biên, basePrice `0.0001` hoặc giá dương; `taxRate=0` | API trả `202 Queued` sau publisher confirm; worker tạo product và operation trong cùng tenant; `DataCreated` chứa snapshot/version/sequence; basePrice phải `> 0`; tax giữ 0 |
| TA-003 | Tạo product thiếu/null/rỗng sku/name/unit/basePrice/taxRate; SKU/name/unit vượt max; UUID category sai; gửi field thừa | Thiếu trả `ERR_REQUIRED_FIELD`; vượt max/sai kiểu `ERR_INVALID_FIELD`; UUID sai `ERR_INVALID_UUID`; field thừa `ERR_UNKNOWN_FIELD`; không tạo product |
| TA-004 | basePrice `0`, âm, overflow hoặc precision >4 | Zero/âm trả `422 ERR_PRICE_MUST_BE_POSITIVE`; overflow/precision sai trả `400 ERR_INVALID_DECIMAL`; không tạo/cập nhật product |
| TA-005 | Tạo product trùng SKU/barcode trong cùng tenant; trùng giữa hai tenant | API trả `202`; worker gửi `CommandRejected` với `ERR_DUPLICATE_SKU` cho cùng tenant/user; tenant khác được phép nếu unique key có tenant scope |
| TA-006 | Sửa product với expectedVersion hiện tại/stale/thiếu; cập nhật giá/name/unit; gửi `tenantId`, status, version giả hoặc field lạ | Giá mới phải dương; stale `409 ERR_VERSION_CONFLICT`; thiếu `ERR_REQUIRED_FIELD`; field lạ/cấm `ERR_UNKNOWN_FIELD`; tax giữ 0 trong v1 |
| TA-007 | Deactivate product chưa dùng/đã có invoice; thử dùng inactive trong invoice mới | Deactivate giữ history; invoice mới dùng product inactive trả `422 ERR_PRODUCT_INACTIVE` |
| TA-008 | Xóa product chưa được tham chiếu/đã có history; expectedVersion hợp lệ/sai | Chưa có history được hard-delete; đã dùng trả `409 ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION` và không mất history |
| TA-009 | Bulk price với operation Increase/Decrease, value dương, 1 item và batch limit; danh sách rỗng, UUID sai, ID tenant B, field thừa | Request hợp lệ cập nhật transactionally đúng tenant và tăng version; empty `ERR_INVALID_FIELD`, UUID sai `ERR_INVALID_UUID`, ID tenant B `404 ERR_PRODUCT_NOT_FOUND`, field thừa `ERR_UNKNOWN_FIELD`; batch không cập nhật một phần |
| TA-010 | Bulk price operation/mode sai; value `0`, âm, dương; expectedVersions thiếu/stale; kết quả decrease làm giá <=0 hoặc overflow | operation/mode sai `ERR_INVALID_ENUM`; value <=0 `422 ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE`; stale `409 ERR_VERSION_CONFLICT`; giá sau điều chỉnh <=0 `ERR_PRICE_MUST_BE_POSITIVE`; overflow/precision sai `ERR_INVALID_DECIMAL`; rollback toàn batch |
| TA-011 | Liệt kê/đọc Staff, lọc inactive, page/search ở biên | Chỉ account Staff thuộc tenant hiện tại; inactive vẫn xem được; response không chứa credential |
| TA-012 | Tạo Staff với username/displayName/password/email/phone ở biên; username duplicate | Valid trả `201`; password 12–128 ký tự; email/phone theo schema; duplicate trả `409 ERR_DUPLICATE_USERNAME` |
| TA-013 | Tạo Staff thiếu/null/rỗng required field, email sai, password ngoài range, gửi role/tenant/status hoặc field lạ | Required `ERR_REQUIRED_FIELD`; email `ERR_INVALID_EMAIL`; password range `ERR_INVALID_FIELD`; field lạ `ERR_UNKNOWN_FIELD`; không tạo account |
| TA-014 | Sửa Staff với expectedVersion current/stale/thiếu; gửi credential, tenantId, role, status | Chỉ username/displayName/email/phone được sửa; version cũ `409 ERR_VERSION_CONFLICT`; field thừa `400 ERR_UNKNOWN_FIELD` |
| TA-015 | Deactivate Staff có nhiều session/device | Status Inactive, revoke mọi session/device ngay, giữ lịch sử/audit |
| TA-016 | Xóa Staff chưa dùng và đã có invoice/history/device/session | Chưa có dependency được hard-delete; có dependency trả `409 ERR_STAFF_DELETE_REQUIRES_DEACTIVATION` |
| TA-017 | Tìm/xem/tạo customer; fullName/code/email/phone/address hợp lệ và tại biên | Create trả `202 Queued`; worker chỉ commit trong tenant JWT, phát snapshot sau commit; email sai `ERR_INVALID_EMAIL`; field vượt max `ERR_INVALID_FIELD`; duplicate code phát `CommandRejected`/`ERR_DUPLICATE_CUSTOMER_CODE` |
| TA-018 | Tạo/sửa customer thiếu code/fullName hoặc expectedVersion; gửi `tenantId`/field ngoài DTO | Thiếu `ERR_REQUIRED_FIELD`; field thừa `ERR_UNKNOWN_FIELD`; không tạo/cập nhật sai tenant |
| TA-019 | Xóa customer chưa được invoice tham chiếu và customer có invoice history; thử deactivate customer đã dùng | Customer chưa tham chiếu được hard-delete; customer có history trả `409 ERR_CUSTOMER_DELETE_REQUIRES_DEACTIVATION`; deactivate giữ nguyên invoice FK và khiến customer không dùng được cho invoice mới |
| TA-020 | Cập nhật `StoreProfile`, `ReceiptTemplate`, `ProductDisplay` hợp lệ; đọc lại `/settings/effective` | Value lưu đúng tenant, version tăng; cấu hình chỉ ảnh hưởng hiển thị/in; không đổi permission, giá hoặc invoice |
| TA-021 | Setting thiếu `value`/`expectedVersion`, version stale, sai kiểu, template/layout/sortBy enum lạ, setting key không tồn tại, field thừa | Thiếu `ERR_REQUIRED_FIELD`; stale `ERR_VERSION_CONFLICT`; kiểu sai `ERR_INVALID_FIELD`; enum lạ `ERR_INVALID_ENUM`; key lạ `ERR_SETTING_NOT_FOUND`; field thừa `ERR_UNKNOWN_FIELD` |
| TA-022 | Cập nhật Tenant Feature Flag true/false và gọi endpoint global | TenantAdmin đổi tenant override được; không thể đổi global; effective config theo thứ tự Tenant override > Global > default |
| TA-023 | Reverse invoice Sale đã sync hợp lệ, invoice không tồn tại, đã reverse, hoặc reverse đồng thời từ hai request | Tạo đúng một invoice âm mới; invoice gốc không bị sửa/xóa nội dung; ID lạ trả `404 ERR_INVOICE_NOT_FOUND`; reverse lần hai trả `409 ERR_INVOICE_ALREADY_REVERSED`; transaction/outbox nhất quán |
| TA-024 | Correction invoice hợp lệ với detail mới; thiếu reason/details, details rỗng, quantity/amount sai | Hợp lệ reverse invoice cũ rồi tạo Correction mới atomically; thiếu field `ERR_REQUIRED_FIELD`, details rỗng `ERR_INVALID_FIELD`, quantity sai `ERR_INVALID_QUANTITY`, giá sai `ERR_PRICE_MUST_BE_POSITIVE`, total lệch `ERR_INVOICE_TOTAL_INVALID`; không để lại reverse một phần |
| TA-025 | TenantAdmin truy cập product/customer/staff/invoice của tenant B hoặc endpoint platform | Không đọc/sửa chéo tenant; trả 404/403 theo policy; DB tenant B không đổi |
| TA-026 | Dùng quyền Staff được kế thừa: TenantAdmin tạo Sale invoice, đọc invoice và in trên thiết bị client | Thành công theo cùng quy tắc sync, snapshot và print của Staff; không có quyền bỏ qua validation/business rule |

## 5. Staff

| ID | Chức năng / dữ liệu | Kết quả mong đợi |
|---|---|---|
| ST-001 | Xem/tìm kiếm product active theo tên/SKU/barcode; search rỗng và ở độ dài biên | Chỉ product active của tenant; response không cho sửa basePrice |
| ST-002 | Yêu cầu `includeInactive=true`, sửa giá, deactivate/delete/create product | Không được thao tác quản lý product; mutation trả `403 ERR_ACCESS_DENIED`; product không đổi |
| ST-003 | Tìm kiếm/lấy customer trong tenant; customer có/không phone | Chỉ customer cùng tenant; phone hiển thị theo quyền/API contract |
| ST-004 | Tạo/sửa/xóa customer bằng Staff, gồm request có `tenantId`/role giả mạo | Từ chối `403 ERR_ACCESS_DENIED`; không tạo hoặc thay đổi customer; request không thể vượt qua RBAC bằng field bổ sung |
| ST-005 | Tạo Sale invoice với product active, quantity `0.0001`/dương, unitPrice `0.0001`/dương, tax fields 0, tổng khớp | API trả `202 Queued`; worker ghi invoice/operation/change/outbox atomically; client chỉ mark Synced sau `DataCreated` hoặc pull thấy cùng operation/entity; retry không duplicate |
| ST-006 | Sale quantity bằng 0/âm; unitPrice bằng 0/âm; total không dương; decimal overflow/precision sai | API trả `202` sau khi queue nhận; worker phát `CommandRejected` với `ERR_INVALID_QUANTITY`, `ERR_PRICE_MUST_BE_POSITIVE`, `ERR_INVOICE_TOTAL_INVALID` hoặc `ERR_INVALID_DECIMAL`; không commit invoice/change |
| ST-007 | Invoice total khớp/lệch subtotal, detail, discount và rounding; tax fields luôn bằng 0 | Tổng hợp lệ phát `DataCreated`; tổng lệch phát `CommandRejected`/`ERR_INVOICE_TOTAL_INVALID`; không chạy test tính/áp thuế trong v1 |
| ST-008 | Sync invoice thiếu/null required field; details rỗng; enum/UUID/currency sai; gửi field thừa ở root hoặc nested detail | Thiếu `ERR_REQUIRED_FIELD`; details rỗng/sai kiểu `ERR_INVALID_FIELD`; enum `ERR_INVALID_ENUM`; UUID `ERR_INVALID_UUID`; currency sai `ERR_INVALID_FIELD`; field thừa `ERR_UNKNOWN_FIELD`; không commit partial data |
| ST-009 | Gửi payload dư `tenantId`, `role`, status, syncedAt hoặc snapshot thuộc product khác | Client không thể mass-assign tenant/actor; lỗi business phát `CommandRejected` với `ERR_PRODUCT_NOT_FOUND`/`ERR_INVALID_FIELD`; không ghi invoice |
| ST-010 | Sync cùng operation ID + cùng payload; cùng operation ID + payload khác | Retry cùng payload không duplicate; payload khác được worker ghi nhận và phát `CommandRejected`/`ERR_SYNC_PAYLOAD_MISMATCH` |
| ST-011 | Sale dùng customer không phone khi `RequireCustomerPhone=true`; product inactive/not found; tenant inactive/device revoked | Worker phát `CommandRejected` với `ERR_CUSTOMER_PHONE_REQUIRED`, `ERR_PRODUCT_INACTIVE`, `ERR_PRODUCT_NOT_FOUND`, `ERR_TENANT_INACTIVE` hoặc `ERR_DEVICE_REVOKED`; không ghi invoice; khi flag false, phone không bắt buộc |
| ST-012 | Xem invoice thuộc tenant; truy cập invoice tenant B/ID không tồn tại | Invoice cùng tenant trả snapshot; invoice không tồn tại/ngoài tenant trả `404 ERR_INVOICE_NOT_FOUND`, không rò dữ liệu |
| ST-013 | In invoice trên printer local thành công, timeout, Bluetooth bật/tắt | In không thay đổi invoice; lỗi dùng `ERR_PRINT_FAILED`, invoice không rollback; khi Bluetooth flag tắt không gọi Bluetooth printer |
| ST-014 | Staff gọi reverse bằng request hợp lệ, rồi thử gửi thêm `adminPin`/`adminUserId`; thử correction | Request reverse hợp lệ trả `403 ERR_ACCESS_DENIED`; payload dư field trả `400 ERR_UNKNOWN_FIELD`; correction trả `403 ERR_ACCESS_DENIED`; không tạo invoice hoặc outbox |
| ST-015 | Staff gọi CRUD account, settings, tenant/global flags, bulk price, product/customer mutation | `403 ERR_ACCESS_DENIED`; không có thay đổi |
| ST-016 | Staff gọi invoice sync khi RabbitMQ confirm thành công/thất bại; hai thiết bị Staff cùng tenant đang kết nối SignalR | Confirm: `202 Queued`; broker outage trước confirm: `503 ERR_SERVICE_UNAVAILABLE`; sau commit cả client cùng tenant nhận đúng invoice snapshot/sequence, client khác tenant không nhận; danh sách chèn bản ghi mới lên đầu không cần GET refresh |

## 6. Kiểm tra xuyên suốt

| ID | Kịch bản | Kết quả mong đợi |
|---|---|---|
| X-001 | Không có JWT, JWT hết hạn, role không hợp lệ, account/device bị revoke | `401` cho unauthenticated/revoked; role thiếu quyền `403`; mọi response theo error envelope |
| X-002 | Mọi query/command gửi `X-Tenant-Id` của tenant khác hoặc nhét `tenantId` vào body | Server bỏ qua/reject field giả mạo; scope chỉ lấy từ JWT/path được SuperAdmin kiểm soát |
| X-003 | Hai TenantAdmin cập nhật cùng resource với expectedVersion giống nhau | Chỉ một update thắng; request stale trả `409 ERR_VERSION_CONFLICT`; không lost update |
| X-004 | Kiểm tra response/error/log sau create account và auth operations | Không lộ plaintext password, hash, PIN, refresh token hoặc secret |
| X-005 | Kiểm tra toàn bộ error case | Đúng HTTP status/error code catalog, `data: null`, correlation ID; không hardcode message nghiệp vụ |

## 7. Quy tắc đã chốt

1. Thuế chưa thuộc phạm vi v1; giữ `taxRate`/`taxAmount` bằng 0 để tương thích schema, chuyển xử lý thuế sang phần mở rộng sau.
2. Giá sản phẩm và `unitPrice` phải lớn hơn 0. API chỉ nhận giá trị dương; UI tự quyết định cách hiển thị dấu. Bulk price gửi độ lớn dương và field `operation` riêng (`Increase`/`Decrease`).
3. Reject mọi JSON field không khai báo, kể cả field lồng nhau, bằng `400 ERR_UNKNOWN_FIELD`; không ignore field lạ hoặc mass-assign.
4. Mỗi nhóm lỗi có error code ổn định và cụ thể: thiếu field `ERR_REQUIRED_FIELD`; field lạ `ERR_UNKNOWN_FIELD`; sai kiểu/range chung `ERR_INVALID_FIELD`; UUID/email/timezone sai dùng `ERR_INVALID_UUID`/`ERR_INVALID_EMAIL`/`ERR_INVALID_TIME_ZONE`; enum/decimal/paging dùng `ERR_INVALID_ENUM`/`ERR_INVALID_DECIMAL`/`ERR_INVALID_PAGING`; giá không dương dùng `ERR_PRICE_MUST_BE_POSITIVE`; bulk magnitude không dương dùng `ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE`.

## 8. Điều kiện hoàn tất bộ test

- Mọi test case trong các bảng role chạy trên API contract/integration suite với tối thiểu Tenant A và Tenant B.
- Case marked **Blocked/TBD** được chuyển thành quyết định business/API trước khi assertion được dùng làm release gate.
- Có test chứng minh quyền bị từ chối không tạo DB, audit, outbox hoặc printer side effect ngoài quy định.
- Regression gate gồm RBAC, tenant isolation, invoice immutability/reverse, feature flags và error envelope.
