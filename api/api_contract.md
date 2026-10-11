# API CONTRACT - POS SaaS

**Version:** v1  
**Base URL:** `/api/v1`  
**Format:** JSON UTF-8  
**Authentication:** `Authorization: Bearer <JWT>`  
**Database:** MySQL 8 + EF Core 8

## 1. Quy ước chung

### Headers

| Header | Bắt buộc | Mô tả |
|---|---:|---|
| `Authorization` | Có, trừ login/version | JWT |
| `X-Tenant-Id` | Không | Không được tin giá trị này; server lấy tenant từ token |
| `X-Device-Id` | Không | Client tự gửi ID ổn định; server tự sinh ID nếu thiếu |
| `X-Platform` | Không | Platform client; có thể suy ra từ User-Agent |
| `X-App-Version` | Không | Version app; có thể suy ra từ User-Agent |
| `X-Correlation-Id` | Khuyến nghị | Truy vết request |
| `Accept-Language` | Không | `vi` hoặc `en`; mặc định `vi` |
| `Idempotency-Key` | Có với command retry | Nên bằng `ClientOperationId` |

Backend localization middleware chọn culture từ `Accept-Language` và trả `Content-Language`. Backend không dịch thông báo nghiệp vụ; mobile vẫn dịch `error.code` theo locale người dùng.

Correlation middleware chấp nhận `X-Correlation-Id` dạng UUID hợp lệ hoặc tự tạo UUID mới, lưu cho request, rồi trả lại trong response header và `meta.correlationId`. MVC result filter chuẩn hóa response thành công/lỗi; exception middleware ánh xạ exception và HTTP error không có body sang error envelope.

### Response thành công

```json
{
  "data": {},
  "meta": {
    "correlationId": "uuid",
    "serverTimeUtc": "2026-10-08T10:00:00Z"
  }
}
```

### Response lỗi có hỗ trợ đa ngôn ngữ

API không trả text nghiệp vụ hardcode. Mobile dùng `error.code` để tra `vi.json`/`en.json`.

```json
{
  "data": null,
  "error": {
    "code": "ERR_PRODUCT_NOT_FOUND",
    "args": { "productId": "uuid" },
    "fieldErrors": [],
    "retryable": false
  },
  "meta": {
    "correlationId": "uuid",
    "serverTimeUtc": "2026-10-08T10:00:00Z",
    "apiVersion": "v1"
  }
}
```

Error code là contract ổn định; `message` nếu có chỉ dành cho log kỹ thuật, không dùng để render UI.

### HTTP status

| Status | Ý nghĩa |
|---:|---|
| 200 | Query/update thành công |
| 201 | Tạo resource thành công |
| 202 | Đã nhận để xử lý async |
| 400 | Payload/schema không hợp lệ |
| 401 | JWT hết hạn, device revoked hoặc chưa đăng nhập |
| 403 | Không đủ role/quyền |
| 404 | Resource không tồn tại trong tenant |
| 409 | Conflict, duplicate hoặc version cũ |
| 422 | Vi phạm business rule |
| 429 | Rate limit |
| 500 | Lỗi không xác định, không lộ chi tiết |

## 2. Authentication và device

### `POST /auth/login`

**Role:** public

Request:

```json
{
  "username": "cashier01",
  "password": "secret"
}
```

Login chỉ nhận username/password. Server tìm user theo username trên mọi tenant và xác định tenant từ tài khoản có password khớp. Nếu cùng username/password khớp nhiều tài khoản, server từ chối để tránh chọn nhầm tenant. Client có thể tự gửi `X-Device-Id`, `X-Platform`, `X-App-Version`; nếu thiếu, server lấy platform/version từ User-Agent và tự sinh device ID.

Response 200:

```json
{
  "data": {
    "accessToken": "jwt",
    "accessTokenExpiresAtUtc": "2026-10-08T10:15:00Z",
    "refreshToken": "opaque-token",
    "user": {
      "id": "uuid",
      "tenantId": "uuid-or-null",
      "role": "Staff",
      "displayName": "Cashier"
    },
    "device": { "id": "uuid", "isRevoked": false }
  }
}
```

Errors: `ERR_INVALID_CREDENTIALS`, `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_FORCE_UPDATE_REQUIRED`.

### `POST /auth/refresh`

Request: `{ "refreshToken": "opaque-token" }`. Tenant user được kiểm tra trong `user_devices`; SuperAdmin được kiểm tra trong `platform_admin_sessions`. Cả hai đều kiểm tra trạng thái account/session và rotate token; không lưu plaintext refresh token. Refresh token cũ dùng lại sẽ bị từ chối.

### `POST /auth/logout`

Thu hồi session hiện tại. Mọi request có access token thuộc session bị revoke sẽ bị từ chối ngay; không xóa dữ liệu server.

Access token sống 15 phút; refresh token mặc định sống 30 ngày. Login và refresh bị giới hạn 10 request/phút/IP. Password dùng BCrypt; secret ký JWT lấy từ `JWT_SECRET_KEY` và tối thiểu 32 byte.

## 3. Products, categories và customers

TenantAdmin có quyền quản lý sản phẩm trong tenant; Staff chỉ được tìm/xem sản phẩm active. Mọi thao tác lấy tenant từ JWT, không nhận tenant scope do client quyết định.

### `GET /products?search=&page=&pageSize=&updatedSince=&includeInactive=`

**Role:** Staff, TenantAdmin.  
Staff chỉ nhận sản phẩm active. TenantAdmin có thể đặt `includeInactive=true` để xem cả sản phẩm đã vô hiệu hóa. `updatedSince` dùng để pull incremental.

```json
{
  "data": [{
    "id": "uuid",
    "sku": "SKU-001",
    "barcode": "893...",
    "name": "Coffee",
    "unit": "cup",
    "basePrice": 25000.0000,
    "taxRate": 0,
    "version": 4,
    "updatedAtUtc": "2026-10-08T09:00:00Z"
  }],
  "meta": {
    "paging": {
      "nextCursor": "cursor",
      "hasMore": true,
      "pageSize": 50
    }
  }
}
```

### `POST /products`

**Role:** TenantAdmin. Request yêu cầu `sku`, `name`, `unit`, `basePrice`, `taxRate`.

Errors: `ERR_ACCESS_DENIED`, `ERR_DUPLICATE_SKU`, `ERR_DUPLICATE_BARCODE`.

### `PATCH /products/{id}`

**Role:** TenantAdmin. Có `expectedVersion`; nếu version không khớp trả `409 ERR_SYNC_CONFLICT`. Staff không được sửa giá gốc.

### `POST /products/{id}/deactivate`

**Role:** TenantAdmin. Vô hiệu hóa sản phẩm, giữ nguyên dữ liệu lịch sử; không cho invoice mới sử dụng sản phẩm inactive. Request có `expectedVersion`; trả Product đã cập nhật.

### `DELETE /products/{id}?expectedVersion=`

**Role:** TenantAdmin. Chỉ hard-delete sản phẩm chưa được tham chiếu bởi invoice hoặc dữ liệu lịch sử. Nếu đã được sử dụng, trả `409 ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION`; hãy dùng endpoint deactivate. Response `200` là `DeletionResult` trong success envelope.

### `POST /products/bulk-price`

**Role:** TenantAdmin. Request:

```json
{
  "productIds": ["uuid"],
  "mode": "Percentage",
  "value": 5.0,
  "expectedVersions": { "product-uuid": 4 }
}
```

Server giới hạn số lượng, lọc theo tenant, transaction toàn bộ và tăng `Version`.

### TenantAdmin quản lý tài khoản Staff

Các endpoint sau chỉ dành cho TenantAdmin và chỉ tác động đến tài khoản role `Staff` thuộc tenant trong JWT. Không nhận `tenantId` hoặc `role` từ request để xác định quyền.

#### `GET /tenant/staff?search=&page=&pageSize=`

Liệt kê Staff của tenant, bao gồm tài khoản inactive; response có paging và không chứa credential.

#### `POST /tenant/staff`

Tạo tài khoản Staff. Request gồm `username`, `initialPassword`, `displayName`, `email`, `phone`. Username duy nhất trong tenant; password chỉ dùng một lần lúc tạo, không trả về và không ghi log. Response `201` trả `StaffAccount`.

#### `GET /tenant/staff/{userId}`

Đọc tài khoản Staff trong tenant hiện tại; user ngoài tenant hoặc không phải Staff trả `404 ERR_STAFF_NOT_FOUND`.

#### `PATCH /tenant/staff/{userId}`

Cập nhật `username`, `displayName`, `email`, `phone`; bắt buộc `expectedVersion`. Không nhận tenant, role, status hoặc credential. Version cũ trả `409 ERR_VERSION_CONFLICT`.

#### `POST /tenant/staff/{userId}/deactivate`

Vô hiệu hóa Staff và revoke toàn bộ device/session của user ngay lập tức. Dữ liệu lịch sử được giữ lại. Response trả `StaffAccount` với status `Inactive`.

#### `DELETE /tenant/staff/{userId}`

Hard-delete chỉ khi Staff chưa được dùng và không có dữ liệu nghiệp vụ/device/session cần giữ. Nếu có lịch sử/dependency, trả `409 ERR_STAFF_DELETE_REQUIRES_DEACTIVATION`; dùng deactivate để giữ audit và tham chiếu lịch sử. Response `200` trả `{ "id": "uuid", "deleted": true }` trong envelope.

`StaffAccount` gồm `id`, `tenantId`, `username`, `displayName`, `email`, `phone`, `status` (`Active`/`Inactive`), `version`, `createdAtUtc`, `updatedAtUtc`; không gồm password/hash, PIN hoặc token.

Errors: `ERR_STAFF_NOT_FOUND`, `ERR_DUPLICATE_USERNAME`, `ERR_VERSION_CONFLICT`, `ERR_STAFF_DELETE_REQUIRES_DEACTIVATION`.

### `GET /customers?search=&updatedSince=` và `POST /customers`

Staff được tìm kiếm/tạo theo chính sách tenant; TenantAdmin có CRUD đầy đủ. Nếu feature `RequireCustomerPhone` bật, invoice có customer bắt buộc phone.

## 4. Invoice

### `POST /invoices/sync`

**Role:** Staff, TenantAdmin. Đây là command idempotent và là endpoint chính cho offline outbox.

Request:

```json
{
  "clientOperationId": "uuid",
  "invoice": {
    "id": "uuid",
    "invoiceNumber": "LOCAL-0001",
    "invoiceType": "Sale",
    "customerId": "uuid",
    "clientCreatedAtUtc": "2026-10-08T09:00:00Z",
    "currencyCode": "VND",
    "subtotal": 100000,
    "discountAmount": 0,
    "taxAmount": 0,
    "roundingAmount": 0,
    "totalAmount": 100000,
    "details": [{
      "lineNumber": 1,
      "productId": "uuid",
      "skuSnapshot": "SKU-001",
      "productNameSnapshot": "Coffee",
      "unitSnapshot": "cup",
      "quantity": 2,
      "unitPrice": 50000,
      "discountAmount": 0,
      "taxRate": 0,
      "taxAmount": 0,
      "lineTotal": 100000
    }]
  }
}
```

Response 201 khi insert mới:

```json
{
  "data": {
    "operationId": "uuid",
    "status": "Applied",
    "created": true,
    "resource": {
      "invoiceId": "uuid",
      "invoiceNumber": "INV-000001",
      "status": "Synced",
      "syncedAtUtc": "2026-10-08T09:01:00Z"
    }
  }
}
```

Retry cùng `clientOperationId` trả kết quả cũ, không tạo invoice mới. Cùng operation id nhưng payload hash khác trả `409 ERR_SYNC_PAYLOAD_MISMATCH`.

Errors: `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_PRODUCT_NOT_FOUND`, `ERR_PRODUCT_INACTIVE`, `ERR_INVOICE_TOTAL_INVALID`, `ERR_CUSTOMER_PHONE_REQUIRED`, `ERR_SYNC_CONFLICT`.

### `GET /invoices/{id}`

**Role:** Staff, TenantAdmin. Chỉ đọc invoice cùng tenant, gồm detail snapshot và thông tin reverse.

Staff và TenantAdmin được in invoice đã đọc trên thiết bị client. App dựng bản in từ invoice snapshot và `ReceiptTemplate` trong `/settings/effective`, rồi gửi tới máy in local/Bluetooth; in không thay đổi invoice và không có server-side print endpoint. Nếu in lỗi, mobile dùng `ERR_PRINT_FAILED`; invoice không rollback.

### `POST /invoices/{id}/reverse`

**Role:** TenantAdmin hoặc Staff kèm PIN hợp lệ. Request:

```json
{ "adminUserId": "uuid", "adminPin": "******", "reasonCode": "CUSTOMER_CANCELLED" }
```

Server lock invoice gốc, kiểm tra chưa reverse, tạo invoice âm mới và outbox event trong cùng transaction. Không update/delete detail gốc.

Errors: `ERR_INVOICE_NOT_FOUND`, `ERR_INVOICE_ALREADY_REVERSED`, `ERR_INVOICE_IMMUTABLE`, `ERR_ADMIN_PIN_INVALID`.

### `POST /invoices/{id}/correction`

**Role:** TenantAdmin. Thực hiện reverse invoice cũ rồi tạo invoice correction mới; request chứa detail mới và lý do. Không mutate invoice cũ.

## 5. Sync pull và feature flags

### `GET /sync/pull?cursor=&limit=`

**Role:** Staff, TenantAdmin. Trả changes của đúng tenant sau cursor, gồm products, categories, customers, invoices và `nextCursor`. Cursor opaque, không cho client tự đoán.

### `GET /settings/effective`

**Role:** authenticated tenant user. Trả cấu hình sau khi hợp nhất global settings và tenant override.

```json
{
  "data": {
    "RequireCustomerPhone": true,
    "EnableInventory": false,
    "EnableBluetoothPrint": true,
    "AutoRoundMoney": true,
    "storeProfile": {
      "storeName": "Panxii Store",
      "address": null,
      "phone": null,
      "taxCode": null
    },
    "receiptTemplate": {
      "templateId": "Standard",
      "showStoreAddress": true,
      "showTaxBreakdown": true,
      "footerText": null
    },
    "productDisplay": {
      "layout": "Grid",
      "sortBy": "Name",
      "showSku": true,
      "showBarcode": false,
      "showCategory": true
    },
    "settingsVersion": 12
  }
}
```

### `PUT /settings/{key}`

**Role:** TenantAdmin với tenant setting. Giá trị phải validate theo schema của key. Các key cấu hình cửa hàng gồm:

- `StoreProfile`: `{ "storeName": "...", "address": null, "phone": null, "taxCode": null }`.
- `ReceiptTemplate`: `{ "templateId": "Standard", "showStoreAddress": true, "showTaxBreakdown": true, "footerText": null }`; `templateId` chỉ nhận template được app hỗ trợ.
- `ProductDisplay`: `{ "layout": "Grid", "sortBy": "Name", "showSku": true, "showBarcode": false, "showCategory": true }`; `layout` là `Grid`/`List`, `sortBy` là `Name`/`Sku`.

Request dùng `{ "value": <value theo schema>, "expectedVersion": 12 }`. Thay đổi áp dụng cho tenant hiện tại và được trả về bởi `/settings/effective`. Các tùy chọn này chỉ ảnh hưởng trình bày/in, không thay đổi quyền, giá hoặc dữ liệu invoice. SuperAdmin quản lý global feature flags qua `/platform/feature-flags`.

### SuperAdmin quản lý tenant và onboarding TenantAdmin

Chỉ SuperAdmin được quản lý tenant. Tạo tenant và tài khoản TenantAdmin đầu tiên là một use case/transaction: nếu tạo account thất bại, tenant cũng không được lưu. Password chỉ được hash, không trả về hoặc ghi log. Các API quản lý TenantAdmin ở `/platform/tenants/{tenantId}/admins` dùng để quản lý thêm account sau onboarding.

#### `GET /platform/tenants?search=&page=&pageSize=&includeDeleted=`

Liệt kê tenant, mặc định không gồm tenant đã soft-delete. `includeDeleted=true` chỉ dành cho SuperAdmin. Response có paging chuẩn.

#### `POST /platform/tenants`

Tạo cửa hàng và TenantAdmin đầu tiên. Request:

```json
{
  "code": "CAFE-001",
  "name": "Cafe Store",
  "timeZone": "Asia/Ho_Chi_Minh",
  "currencyCode": "VND",
  "tenantAdmin": {
    "username": "cafe.admin",
    "initialPassword": "one-time-secret",
    "displayName": "Cafe Owner",
    "email": "owner@example.com",
    "phone": "0900000000"
  }
}
```

`code` duy nhất toàn hệ thống; username duy nhất trong tenant mới. Response `201` trả `{ tenant, tenantAdmin }`, không chứa credential. Tenant mới có `status: Active`, version ban đầu `1`.

#### `GET /platform/tenants/{tenantId}?includeDeleted=`

Đọc thông tin tenant. Tenant đã soft-delete chỉ được đọc khi `includeDeleted=true`.

#### `PATCH /platform/tenants/{tenantId}`

Cập nhật `code`, `name`, `timeZone`, `currencyCode`. Không nhận status hoặc `isDeleted`; dùng lifecycle endpoints bên dưới. Bắt buộc `expectedVersion`; mismatch trả `409 ERR_VERSION_CONFLICT`.

#### `POST /platform/tenants/{tenantId}/suspend`

Đặt `status: Suspended`, revoke toàn bộ tenant user devices/sessions để các token hiện hành mất hiệu lực.

#### `POST /platform/tenants/{tenantId}/reactivate`

Đặt tenant chưa bị xóa về `status: Active`. Các session đã revoke không được khôi phục; users cần đăng nhập lại.

#### `DELETE /platform/tenants/{tenantId}`

Soft-delete tenant: đặt `isDeleted: true`, `status: Closed`, revoke toàn bộ devices/sessions và ghi audit. Không xóa vật lý tenant, users hoặc dữ liệu nghiệp vụ. Tenant đã xóa không thể reactivate.

`PlatformTenant` gồm `id`, `code`, `name`, `status` (`Active`/`Suspended`/`Closed`), `timeZone`, `currencyCode`, `isDeleted`, `version`, `createdAtUtc`, `updatedAtUtc`.

Errors: `ERR_TENANT_NOT_FOUND`, `ERR_DUPLICATE_TENANT_CODE`, `ERR_INVALID_REQUEST`, `ERR_INVALID_PAGING`, `ERR_VERSION_CONFLICT`.

### SuperAdmin quản lý tài khoản TenantAdmin

Các endpoint dưới đây chỉ dành cho **SuperAdmin**. `{tenantId}` xác định tenant đích; server phải kiểm tra tenant tồn tại và tenant scope trên mọi thao tác. Không cho client thay đổi role: tài khoản do nhóm API này quản lý luôn có role `TenantAdmin`.

#### `GET /platform/tenants/{tenantId}/admins?search=&page=&pageSize=`

Liệt kê tài khoản TenantAdmin của tenant, gồm cả tài khoản đã vô hiệu hóa. Response dùng collection envelope và paging chuẩn; không trả password, password hash, PIN hoặc token.

#### `POST /platform/tenants/{tenantId}/admins`

Tạo tài khoản TenantAdmin trong tenant. Request:

```json
{
  "username": "store.admin",
  "initialPassword": "one-time-secret",
  "displayName": "Store Admin",
  "email": "admin@example.com",
  "phone": "0900000000"
}
```

`username` phải duy nhất trong tenant. `initialPassword` chỉ dùng lúc tạo và tuyệt đối không được trả lại hoặc ghi log. Response `201` trả resource `TenantAdminAccount` và không chứa credential.

#### `GET /platform/tenants/{tenantId}/admins/{userId}`

Đọc chi tiết tài khoản trong đúng tenant; tài khoản không thuộc tenant trả `404`.

#### `PATCH /platform/tenants/{tenantId}/admins/{userId}`

Cập nhật thông tin được phép của tài khoản, không nhận `tenantId`, `role`, status hoặc credential. Bắt buộc `expectedVersion`; version cũ trả `409 ERR_VERSION_CONFLICT`.

```json
{
  "username": "store.admin",
  "displayName": "Store Administrator",
  "email": "admin@example.com",
  "phone": "0900000000",
  "expectedVersion": 3
}
```

#### `POST /platform/tenants/{tenantId}/admins/{userId}/deactivate`

Vô hiệu hóa tài khoản. Server phải revoke toàn bộ device/session của user ngay trong cùng thao tác nghiệp vụ; các access token/session đã revoke không được tiếp tục truy cập API. Ghi audit before/after và actor SuperAdmin. Không xóa dữ liệu lịch sử.

#### `DELETE /platform/tenants/{tenantId}/admins/{userId}`

Hard-delete chỉ được phép khi tài khoản chưa được dùng và không có dữ liệu nghiệp vụ/device/session cần giữ. Nếu có lịch sử hoặc dependency, trả `409 ERR_TENANT_ADMIN_DELETE_REQUIRES_DEACTIVATION`; dùng endpoint deactivate để giữ audit và tham chiếu lịch sử. Response `200` dùng `DeletionResult` trong envelope.

```json
{
  "data": { "id": "uuid", "deleted": true },
  "meta": { "correlationId": "uuid", "serverTimeUtc": "2026-10-08T10:00:00Z", "apiVersion": "v1" }
}
```

`TenantAdminAccount` gồm `id`, `tenantId`, `username`, `displayName`, `email`, `phone`, `status` (`Active`/`Inactive`), `version`, `createdAtUtc`, `updatedAtUtc`. Không bao gồm password/hash, PIN, refresh token hoặc secret.

Errors: `ERR_TENANT_NOT_FOUND`, `ERR_TENANT_ADMIN_NOT_FOUND`, `ERR_DUPLICATE_USERNAME`, `ERR_TENANT_ADMIN_DELETE_REQUIRES_DEACTIVATION`, `ERR_VERSION_CONFLICT`.

### SuperAdmin cấu hình global feature flags

#### `GET /platform/feature-flags`

**Role:** SuperAdmin. Liệt kê cấu hình global hiện tại theo registry; mỗi item gồm `key`, `value`, `scope` (`Global`), `version`, `updatedAtUtc`. Collection luôn có `meta.paging` theo quy ước chung.

#### `PUT /platform/feature-flags/{key}`

**Role:** SuperAdmin. Cập nhật giá trị global của flag đã đăng ký. Request:

```json
{
  "value": true,
  "expectedVersion": 12
}
```

Server validate kiểu/range theo flag registry, không chấp nhận key tùy ý; cập nhật version và ghi audit cùng transaction. Client cần đọc lại cấu hình để nhận giá trị mới; backend hiện chưa phát realtime cache-invalidation event. `expectedVersion` bắt buộc để tránh ghi đè thay đổi đồng thời. Tenant override (nếu có) vẫn ưu tiên hơn global theo thứ tự `Tenant override > Global > Application default`.

Errors: `ERR_FEATURE_FLAG_NOT_FOUND`, `ERR_INVALID_REQUEST`, `ERR_VERSION_CONFLICT`.

### `GET /config/version`

**Role:** public. Trả `minimumNativeVersion`, `latestNativeVersion`, `shorebirdPatch`, download URLs. Native version thấp hơn minimum thì mobile chặn login và yêu cầu update.

## 6. Swagger/OpenAPI bắt buộc

- Khai báo Bearer security scheme và các role trong `description`.
- Khai báo request/response schema, enum, nullable, min/max, decimal format.
- Mỗi endpoint có response 400/401/403/404/409/422.
- Dùng `operationId` ổn định, ví dụ `SyncInvoice`, `ReverseInvoice`.
- Dùng `x-error-codes` hoặc component `ErrorResponse` để generator mobile biết error code.
- Không đưa secret, refresh token thật hoặc production URL vào Swagger example.
- Tách Swagger public và internal nếu endpoint SuperAdmin có thông tin nhạy cảm.

## 7. Error code chuẩn

`ERR_ACCESS_DENIED`, `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_INVALID_CREDENTIALS`, `ERR_PRODUCT_NOT_FOUND`, `ERR_PRODUCT_INACTIVE`, `ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION`, `ERR_DUPLICATE_SKU`, `ERR_DUPLICATE_BARCODE`, `ERR_CUSTOMER_PHONE_REQUIRED`, `ERR_INVOICE_NOT_FOUND`, `ERR_INVOICE_IMMUTABLE`, `ERR_INVOICE_ALREADY_REVERSED`, `ERR_INVOICE_TOTAL_INVALID`, `ERR_ADMIN_PIN_INVALID`, `ERR_SYNC_DUPLICATE`, `ERR_SYNC_PAYLOAD_MISMATCH`, `ERR_SYNC_CONFLICT`, `ERR_STAFF_NOT_FOUND`, `ERR_STAFF_DELETE_REQUIRES_DEACTIVATION`, `ERR_DUPLICATE_USERNAME`, `ERR_VERSION_CONFLICT`, `ERR_SETTING_NOT_FOUND`, `ERR_FORCE_UPDATE_REQUIRED`.

Platform admin errors: `ERR_TENANT_NOT_FOUND`, `ERR_DUPLICATE_TENANT_CODE`, `ERR_TENANT_ADMIN_NOT_FOUND`, `ERR_DUPLICATE_USERNAME`, `ERR_TENANT_ADMIN_DELETE_REQUIRES_DEACTIVATION`, `ERR_FEATURE_FLAG_NOT_FOUND`, `ERR_VERSION_CONFLICT`.

Mobile map cùng key trong `vi.json` và `en.json`; không map theo HTTP status đơn thuần.

---

## 8. Canonical API Contract dùng chung

Phần này là chuẩn bắt buộc cho Backend và Mobile. Các endpoint bên trên phải được triển khai theo các DTO và quy tắc ở đây; không tự tạo envelope riêng cho từng endpoint.

### 8.1. Quy tắc JSON

- JSON dùng `camelCase`.
- Tất cả thời gian dùng ISO-8601 UTC, ví dụ `2026-10-08T10:00:00Z`.
- Tất cả ID dùng UUID string chuẩn.
- Tiền và số lượng được truyền dạng JSON number nhưng backend phải parse bằng `decimal`; không dùng `double` trong domain.
- Enum truyền dạng string, không truyền số nguyên. Ví dụ `"Staff"`, `"Synced"`.
- Field không có dữ liệu dùng `null`; không tùy ý đổi giữa `null`, `""` và bỏ field.
- Property mới phải backward-compatible. Không đổi ý nghĩa hoặc kiểu của property đã phát hành.
- Backend bỏ qua field lạ khi đọc request nhưng không được bỏ qua field bắt buộc hoặc sai kiểu.
- Mobile không đọc message để quyết định logic; chỉ dùng `error.code`.

### 8.2. Request metadata chuẩn

Mọi request authenticated dùng các header sau:

```http
Authorization: Bearer <access-token>
Content-Type: application/json
Accept: application/json
X-Correlation-Id: 7a6d2bb8-8c04-4dd5-b4b3-2ce3c8c5e001
X-Device-Id: 4c1d-device-id
```

Các command có thể retry bắt buộc thêm:

```http
Idempotency-Key: 8f14e45f-ea2c-4e8a-a4a8-9d3b7c50e001
```

Quy tắc:

1. `X-Correlation-Id` do client gửi hoặc server tạo nếu thiếu; response phải trả lại.
2. `Idempotency-Key` phải ổn định trong toàn bộ vòng đời retry của cùng một command.
3. Backend lưu hash request theo `(tenantId, idempotencyKey)`. Dùng lại key với payload khác trả `ERR_IDEMPOTENCY_KEY_REUSED`.
4. Không truyền `tenantId` trong body để quyết định tenant. Server lấy tenant từ JWT/device context.

### 8.3. Response envelope thành công

Mọi response thành công đều dùng một trong hai dạng sau.

#### Resource hoặc command đơn

```json
{
  "data": {
    "id": "uuid"
  },
  "meta": {
    "correlationId": "uuid",
    "serverTimeUtc": "2026-10-08T10:00:00Z",
    "apiVersion": "v1"
  }
}
```

#### Collection/paging

```json
{
  "data": [],
  "meta": {
    "correlationId": "uuid",
    "serverTimeUtc": "2026-10-08T10:00:00Z",
    "apiVersion": "v1",
    "paging": {
      "nextCursor": "opaque-cursor-or-null",
      "hasMore": false,
      "pageSize": 50
    }
  }
}
```

Không trả collection trực tiếp dạng `[]`. Nếu endpoint không có dữ liệu, trả `data: []` hoặc `data: null` đúng theo schema endpoint, không trả HTTP 404 cho collection rỗng.

### 8.4. Error envelope chuẩn

Mọi lỗi 4xx/5xx đều dùng cùng schema:

```json
{
  "data": null,
  "error": {
    "code": "ERR_PRODUCT_NOT_FOUND",
    "args": {
      "productId": "uuid"
    },
    "fieldErrors": [
      {
        "field": "invoice.details[0].productId",
        "code": "ERR_PRODUCT_NOT_FOUND",
        "args": { "productId": "uuid" }
      }
    ],
    "retryable": false
  },
  "meta": {
    "correlationId": "uuid",
    "serverTimeUtc": "2026-10-08T10:00:00Z",
    "apiVersion": "v1"
  }
}
```

Quy tắc:

- `error.code` luôn có và là key i18n.
- `error.args` chỉ chứa tham số để mobile render câu dịch; không chứa secret.
- `fieldErrors` dùng cho lỗi validation theo field; có thể là mảng rỗng.
- `retryable` cho biết mobile có được retry tự động. Lỗi nghiệp vụ thường là `false`.
- `data` luôn là `null` khi request thất bại.
- `details` cũ trong các ví dụ trước được chuẩn hóa thành `fieldErrors`; backend không trả đồng thời hai tên.

### 8.5. DTO validation chuẩn

| Quy tắc | Chuẩn |
|---|---|
| UUID | String, format UUID, không nhận empty string |
| String | Có `maxLength`; trim ở server, không trim mã snapshot làm thay đổi dữ liệu |
| Money | `decimal(19,4)`, không NaN/Infinity, kiểm tra range |
| Quantity | `decimal(19,4)`, Sale > 0, Reverse < 0 |
| Percentage | `decimal(9,4)`, range theo nghiệp vụ |
| Page size | Mặc định 50, tối đa 200 |
| Cursor | Opaque string, client không tự tạo hoặc sửa |
| DateTime | UTC ISO-8601; request có timezone khác bị normalize hoặc reject thống nhất |
| Enum | String enum; giá trị không biết trả validation error |

### 8.6. Operation result cho command idempotent

Các command `POST /invoices/sync`, reverse, correction, bulk price và settings update phải trả thêm trạng thái xử lý:

```json
{
  "data": {
    "operationId": "uuid",
    "status": "Applied",
    "resourceId": "uuid",
    "created": true
  },
  "meta": {
    "correlationId": "uuid",
    "serverTimeUtc": "2026-10-08T10:00:00Z",
    "apiVersion": "v1"
  }
}
```

Giá trị `status`: `Applied`, `AlreadyApplied`, `Rejected`. Retry thành công dùng `AlreadyApplied`, nhưng schema response vẫn giữ nguyên.

---

## 9. DTO chuẩn cho mobile và backend

### 9.1. User/session

```json
{
  "id": "uuid",
  "tenantId": "uuid-or-null",
  "role": "Staff",
  "displayName": "Cashier",
  "deviceId": "device-uuid",
  "expiresAtUtc": "2026-10-08T11:00:00Z"
}
```

`password`, `passwordHash`, `adminPin`, refresh token raw và secret tuyệt đối không xuất hiện trong DTO response ngoài token response cần thiết.

### 9.2. Product

```json
{
  "id": "uuid",
  "categoryId": "uuid-or-null",
  "sku": "SKU-001",
  "barcode": "893...",
  "name": "Coffee",
  "unit": "cup",
  "basePrice": 25000.0000,
  "costPrice": null,
  "taxRate": 0.0000,
  "isActive": true,
  "version": 4,
  "updatedAtUtc": "2026-10-08T09:00:00Z"
}
```

### 9.3. Invoice detail và invoice

```json
{
  "lineNumber": 1,
  "productId": "uuid-or-null",
  "skuSnapshot": "SKU-001",
  "productNameSnapshot": "Coffee",
  "unitSnapshot": "cup",
  "quantity": 2.0000,
  "unitPrice": 50000.0000,
  "discountAmount": 0.0000,
  "taxRate": 0.0000,
  "taxAmount": 0.0000,
  "lineTotal": 100000.0000
}
```

```json
{
  "id": "uuid",
  "invoiceNumber": "INV-000001",
  "originalInvoiceId": null,
  "invoiceType": "Sale",
  "status": "Synced",
  "customerId": "uuid-or-null",
  "clientCreatedAtUtc": "2026-10-08T09:00:00Z",
  "syncedAtUtc": "2026-10-08T09:01:00Z",
  "currencyCode": "VND",
  "subtotal": 100000.0000,
  "discountAmount": 0.0000,
  "taxAmount": 0.0000,
  "roundingAmount": 0.0000,
  "totalAmount": 100000.0000,
  "details": []
}
```

### 9.4. Customer, category và settings

```json
{
  "id": "uuid",
  "code": "CUS-001",
  "fullName": "Nguyen Van A",
  "phone": "0900000000",
  "email": null,
  "address": null,
  "isActive": true,
  "version": 1,
  "updatedAtUtc": "2026-10-08T09:00:00Z"
}
```

```json
{
  "settings": {
    "RequireCustomerPhone": true,
    "EnableInventory": false,
    "EnableBluetoothPrint": true,
    "AutoRoundMoney": true
  },
  "settingsVersion": 12,
  "effectiveAtUtc": "2026-10-08T09:00:00Z"
}
```

---

## 10. Bảng endpoint đầy đủ và format bắt buộc

| Method | Endpoint | Request body/query | Response `data` | Status thành công |
|---|---|---|---|---:|
| POST | `/auth/login` | `LoginRequest` | `LoginResponse` | 200 |
| POST | `/auth/refresh` | `RefreshTokenRequest` | `TokenResponse` | 200 |
| POST | `/auth/logout` | Không body | `LogoutResult` | 200 |
| GET | `/products` | query `search,page,pageSize,updatedSince,cursor,includeInactive` | `Product[]` + paging | 200 |
| POST | `/products` | `CreateProductRequest` | `Product` | 201 |
| PATCH | `/products/{id}` | `UpdateProductRequest` + `expectedVersion` | `Product` | 200 |
| POST | `/products/{id}/deactivate` | `expectedVersion` | `Product` | 200 |
| DELETE | `/products/{id}` | query `expectedVersion` | `DeletionResult` | 200 |
| POST | `/products/bulk-price` | `BulkPriceRequest` | `BulkPriceResult` | 200 |
| GET | `/tenant/staff` | query `search,page,pageSize` | `StaffAccount[]` + paging | 200 |
| POST | `/tenant/staff` | `CreateStaffRequest` | `StaffAccount` | 201 |
| GET | `/tenant/staff/{userId}` | path `userId` | `StaffAccount` | 200 |
| PATCH | `/tenant/staff/{userId}` | `UpdateStaffRequest` + `expectedVersion` | `StaffAccount` | 200 |
| POST | `/tenant/staff/{userId}/deactivate` | Không body | `StaffAccount` | 200 |
| DELETE | `/tenant/staff/{userId}` | path `userId` | `DeletionResult` | 200 |
| GET | `/categories` | query filter/cursor | `Category[]` + paging | 200 |
| POST | `/categories` | `CreateCategoryRequest` | `Category` | 201 |
| PATCH | `/categories/{id}` | `UpdateCategoryRequest` + version | `Category` | 200 |
| GET | `/customers` | query `search,page,pageSize,updatedSince,cursor` | `Customer[]` + paging | 200 |
| POST | `/customers` | `CreateCustomerRequest` | `Customer` | 201 |
| PATCH | `/customers/{id}` | `UpdateCustomerRequest` + version | `Customer` | 200 |
| GET | `/invoices/{id}` | path `id` | `Invoice` | 200 |
| POST | `/invoices/sync` | `SyncInvoiceRequest` + idempotency | `OperationResult<Invoice>` | 201/200 |
| POST | `/invoices/{id}/reverse` | `ReverseInvoiceRequest` + idempotency | `OperationResult<Invoice>` | 201 |
| POST | `/invoices/{id}/correction` | `CorrectionInvoiceRequest` + idempotency | `OperationResult<Invoice>` | 201 |
| GET | `/sync/pull` | `cursor,limit` | `SyncChange[]` + paging | 200 |
| GET | `/settings/effective` | Không body | `EffectiveSettings` | 200 |
| PUT | `/settings/{key}` | `UpdateSettingRequest` + version; TenantAdmin tenant scope | `Setting` | 200 |
| GET | `/platform/tenants` | query `search,page,pageSize,includeDeleted` | `PlatformTenant[]` + paging | 200 |
| POST | `/platform/tenants` | `CreatePlatformTenantRequest` including initial `TenantAdmin` | `CreatePlatformTenantResponse` | 201 |
| GET | `/platform/tenants/{tenantId}` | path `tenantId`; optional `includeDeleted` | `PlatformTenant` | 200 |
| PATCH | `/platform/tenants/{tenantId}` | `UpdatePlatformTenantRequest` + `expectedVersion` | `PlatformTenant` | 200 |
| POST | `/platform/tenants/{tenantId}/suspend` | Không body | `PlatformTenant` | 200 |
| POST | `/platform/tenants/{tenantId}/reactivate` | Không body | `PlatformTenant` | 200 |
| DELETE | `/platform/tenants/{tenantId}` | path `tenantId` | `DeletePlatformTenantResponse` (soft-delete) | 200 |
| GET | `/platform/tenants/{tenantId}/admins` | query `search,page,pageSize` | `TenantAdminAccount[]` + paging | 200 |
| POST | `/platform/tenants/{tenantId}/admins` | `CreateTenantAdminRequest` | `TenantAdminAccount` | 201 |
| GET | `/platform/tenants/{tenantId}/admins/{userId}` | path `tenantId,userId` | `TenantAdminAccount` | 200 |
| PATCH | `/platform/tenants/{tenantId}/admins/{userId}` | `UpdateTenantAdminRequest` + `expectedVersion` | `TenantAdminAccount` | 200 |
| POST | `/platform/tenants/{tenantId}/admins/{userId}/deactivate` | Không body | `TenantAdminAccount` | 200 |
| DELETE | `/platform/tenants/{tenantId}/admins/{userId}` | path `tenantId,userId` | `DeletionResult` | 200 |
| GET | `/platform/feature-flags` | Không body | `FeatureFlag[]` + paging | 200 |
| PUT | `/platform/feature-flags/{key}` | `UpdateGlobalFeatureFlagRequest` + version | `FeatureFlag` | 200 |
| GET | `/config/version` | Không body | `VersionConfig` | 200 |

Tất cả endpoint trong bảng đều trả `meta.correlationId`. Endpoint collection luôn trả `meta.paging`, kể cả khi `data` rỗng.

---

## 11. Format request/response chi tiết theo nhóm API

### 11.1. Login

Request `LoginRequest`:

```json
{
  "username": "cashier01",
  "password": "secret"
}
```

Response `LoginResponse`:

```json
{
  "accessToken": "jwt",
  "refreshToken": "opaque-token",
  "accessTokenExpiresAtUtc": "2026-10-08T10:15:00Z",
  "refreshTokenExpiresAtUtc": "2026-11-07T10:00:00Z",
  "user": {},
  "device": { "id": "uuid", "isRevoked": false }
}
```

### 11.2. CRUD Product/Category/Customer

Request create/update phải dùng object riêng, không nhận trực tiếp response object để tránh client gửi nhầm `id`, `tenantId`, audit hoặc `version`.

```json
{
  "sku": "SKU-001",
  "name": "Coffee",
  "unit": "cup",
  "basePrice": 25000.0000,
  "taxRate": 0.0000,
  "categoryId": "uuid-or-null"
}
```

Update bắt buộc gửi `expectedVersion`:

```json
{
  "name": "Coffee large",
  "basePrice": 30000.0000,
  "expectedVersion": 4
}
```

Response luôn là resource đầy đủ theo DTO chuẩn tương ứng, không trả một phần field ngẫu nhiên.

### 11.3. Sync invoice

Request `SyncInvoiceRequest` phải có:

```json
{
  "clientOperationId": "uuid",
  "payloadVersion": 1,
  "invoice": {}
}
```

`clientOperationId` trong body phải trùng `Idempotency-Key`. Nếu khác, trả `400 ERR_IDEMPOTENCY_KEY_MISMATCH`.

Response mới và response retry đều dùng cùng schema:

```json
{
  "operationId": "uuid",
  "status": "Applied",
  "created": true,
  "resource": {},
  "serverCursor": "opaque-cursor"
}
```

Retry trả `status: "AlreadyApplied"`, `created: false`, nhưng vẫn trả cùng `resource` để mobile cập nhật SQLite.

### 11.4. Pull sync

Response:

```json
{
  "changes": [
    {
      "changeType": "Upsert",
      "entityType": "Invoice",
      "entityId": "uuid",
      "version": 1,
      "entity": {}
    }
  ],
  "nextCursor": "opaque-cursor",
  "hasMore": false
}
```

Client chỉ lưu `nextCursor` sau khi toàn bộ `changes` đã upsert vào SQLite thành công. Nếu lỗi giữa batch, gọi lại cursor cũ là an toàn.

---

## 12. Quy tắc tương thích phiên bản

- URL major version thay đổi khi phá vỡ contract: `/api/v2`.
- Thêm field là non-breaking; xóa/đổi kiểu field là breaking.
- Enum mới phải có fallback `Unknown` ở mobile hoặc được phát hành sau mobile.
- Server phải hỗ trợ ít nhất một phiên bản mobile cũ trong thời gian migration đã chốt.
- `payloadVersion` dùng cho request sync; server reject version không hỗ trợ bằng `ERR_UNSUPPORTED_PAYLOAD_VERSION`.
- OpenAPI spec phải commit cùng code backend và được dùng để generate client model hoặc validate contract test.
- Mỗi thay đổi contract cần cập nhật `vi.json`, `en.json`, backend DTO, mobile DTO và contract test trong cùng pull request.

## 13. Error code bổ sung cho contract

| Code | Tình huống |
|---|---|
| `ERR_INVALID_REQUEST` | JSON/schema không hợp lệ |
| `ERR_INVALID_PAGING` | Page hoặc page size không hợp lệ |
| `ERR_UNAUTHENTICATED` | Request chưa xác thực |
| `ERR_NOT_FOUND` | Route/resource không tồn tại |
| `ERR_CONFLICT` | Request xung đột trạng thái hiện tại |
| `ERR_IDEMPOTENCY_KEY_MISMATCH` | Header và body operation id khác nhau |
| `ERR_IDEMPOTENCY_KEY_REUSED` | Một key dùng cho payload khác |
| `ERR_UNSUPPORTED_PAYLOAD_VERSION` | Server không hỗ trợ payload version |
| `ERR_INVALID_ENUM` | Giá trị enum không hợp lệ |
| `ERR_INVALID_DECIMAL` | Tiền/số lượng sai precision hoặc range |
| `ERR_VERSION_CONFLICT` | `expectedVersion` đã cũ |
| `ERR_CURSOR_EXPIRED` | Cursor pull không còn hợp lệ |
| `ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION` | Product có lịch sử/dependency, không thể hard-delete |
| `ERR_STAFF_NOT_FOUND` | Staff không tồn tại trong tenant hiện tại |
| `ERR_STAFF_DELETE_REQUIRES_DEACTIVATION` | Staff có lịch sử/dependency, không thể hard-delete |
| `ERR_DUPLICATE_USERNAME` | Username đã tồn tại trong tenant |
| `ERR_SETTING_NOT_FOUND` | Setting không có trong registry |
| `ERR_TENANT_NOT_FOUND` | Tenant platform đích không tồn tại |
| `ERR_DUPLICATE_TENANT_CODE` | Tenant code đã được sử dụng |
| `ERR_TENANT_ADMIN_NOT_FOUND` | TenantAdmin không tồn tại trong tenant chỉ định |
| `ERR_TENANT_ADMIN_DELETE_REQUIRES_DEACTIVATION` | Tài khoản có lịch sử/dependency, không thể hard-delete |
| `ERR_FEATURE_FLAG_NOT_FOUND` | Feature flag không có trong registry |

## 14. Definition of Done cho một API

Một endpoint chỉ được xem là hoàn tất khi có đủ:

1. Request DTO và response DTO cụ thể.
2. JSON example hợp lệ theo DTO.
3. HTTP status cho success và từng nhóm lỗi.
4. Error code, `args`, `fieldErrors` và `retryable`.
5. Authorization role và tenant scope.
6. OpenAPI `operationId`, schema, enum, nullable, validation.
7. Contract test backend.
8. Mobile model/parser và mapping i18n.
9. Quy tắc retry/idempotency nếu là command.
10. Correlation ID trong response và log.
