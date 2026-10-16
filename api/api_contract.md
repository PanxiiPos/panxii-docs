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
| `Idempotency-Key` | Bắt buộc với command tạo/sync | UUID ổn định qua mọi lần retry |

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

### `GET /me`

**Role:** TenantAdmin, Staff. Trả hồ sơ tài khoản đang xác thực trong tenant hiện tại; không nhận `userId` hoặc `tenantId` từ client.

Response `200` trả `UserProfile`: `id`, `tenantId`, `username`, `displayName`, `email`, `phone`, `role`, `status`, `version`, `createdAtUtc`, `updatedAtUtc`. Không trả password/hash, PIN hoặc token.

```json
{
  "data": {
    "id": "user-uuid",
    "tenantId": "tenant-uuid",
    "username": "cashier01",
    "displayName": "Cashier",
    "email": null,
    "phone": "0900000000",
    "role": "Staff",
    "status": "Active",
    "version": 3,
    "createdAtUtc": "2026-10-08T09:00:00Z",
    "updatedAtUtc": "2026-10-09T09:00:00Z"
  },
  "meta": { "correlationId": "uuid", "serverTimeUtc": "2026-10-09T09:00:00Z" }
}
```

### `PATCH /me`

**Role:** TenantAdmin, Staff. Chỉ cập nhật hồ sơ của chính user từ JWT. Request bắt buộc có `username`, `displayName`, `email`, `phone`, `expectedVersion`; `email` và `phone` có thể là `null` để xóa giá trị. Username duy nhất trong tenant. Version cũ trả `409 ERR_VERSION_CONFLICT`; username trùng trả `409 ERR_DUPLICATE_USERNAME`.

```json
{
  "username": "cashier01",
  "displayName": "Cashier",
  "email": null,
  "phone": "0900000000",
  "expectedVersion": 3
}
```

Password/credential không thuộc API hồ sơ này; không gửi hoặc trả credential. API đổi mật khẩu cần contract riêng nếu được đưa vào phạm vi.

Access token sống 15 phút; refresh token mặc định sống 30 ngày. Login và refresh bị giới hạn 10 request/phút/IP. Password dùng BCrypt; secret ký JWT lấy từ `JWT_SECRET_KEY` và tối thiểu 32 byte.

## 3. Products, categories và customers

TenantAdmin có quyền quản lý sản phẩm trong tenant; Staff chỉ được tìm/xem sản phẩm active. Mọi thao tác lấy tenant từ JWT, không nhận tenant scope do client quyết định.

### `GET /products?search=&cursor=&pageSize=&updatedSince=&includeInactive=`

**Role:** Staff, TenantAdmin.  
Staff chỉ nhận sản phẩm active. TenantAdmin có thể đặt `includeInactive=true` để xem cả sản phẩm đã vô hiệu hóa. `updatedSince` dùng để pull incremental. `search` tìm theo tên, SKU hoặc barcode. Kết quả sắp xếp `name ASC, id ASC`.
Mobile mới dùng cursor keyset opaque: request đầu gửi `cursor=` rỗng, các trang sau truyền nguyên `nextCursor`. `page` chỉ được hỗ trợ như legacy mode cho client API v1 cũ; không dùng trong client mới.

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
    "isActive": true,
    "version": 4,
    "updatedAtUtc": "2026-10-08T09:00:00Z"
  }],
  "meta": {
    "paging": {
      "nextCursor": "opaque-cursor-or-null",
      "hasMore": true,
      "pageSize": 50
    }
  }
}
```

### `POST /products`

**Role:** TenantAdmin. Request yêu cầu `sku`, `name`, `unit`, `basePrice`, `taxRate`; `barcode` và `categoryId` tùy chọn. Barcode tối đa 64 ký tự và duy nhất trong tenant nếu có. `basePrice` phải lớn hơn 0; `taxRate` giữ bằng 0 trong v1, nghiệp vụ thuế là phần mở rộng sau.
Header `Idempotency-Key` là bắt buộc. API publish command bền vững tới RabbitMQ rồi trả `202` với `{ "operationId": "uuid", "status": "Queued" }`. Worker mới tạo product; client nhận event `DataCreated` sau khi MySQL commit.

Errors: `ERR_ACCESS_DENIED`, `ERR_REQUIRED_FIELD`, `ERR_UNKNOWN_FIELD`, `ERR_INVALID_FIELD`, `ERR_INVALID_DECIMAL`, `ERR_PRICE_MUST_BE_POSITIVE`, `ERR_DUPLICATE_SKU`, `ERR_DUPLICATE_BARCODE`.

### `GET /products/{id}`

**Role:** Staff, TenantAdmin. Trả chi tiết product trong tenant hiện tại theo cùng `Product` schema như list. Staff chỉ đọc product active; product inactive hoặc khác tenant trả `404 ERR_PRODUCT_NOT_FOUND`. TenantAdmin đọc được product active/inactive để quản lý.

### `PATCH /products/{id}`

**Role:** TenantAdmin. Cho cập nhật `name`, `barcode`, `unit`, `basePrice`, `taxRate`, `categoryId` và bắt buộc `expectedVersion`. `barcode: null` dùng để xóa barcode; barcode khác null phải duy nhất trong tenant. Nếu version không khớp trả `409 ERR_VERSION_CONFLICT`; barcode trùng trả `409 ERR_DUPLICATE_BARCODE`. Giá mới phải lớn hơn 0. Staff không được sửa giá gốc.

### `POST /products/{id}/deactivate`

**Role:** TenantAdmin. Vô hiệu hóa sản phẩm, giữ nguyên dữ liệu lịch sử; không cho invoice mới sử dụng sản phẩm inactive. Request có `expectedVersion`; trả Product đã cập nhật.

### `DELETE /products/{id}?expectedVersion=`

**Role:** TenantAdmin. Chỉ hard-delete sản phẩm chưa được tham chiếu bởi invoice hoặc dữ liệu lịch sử. Nếu đã được sử dụng, trả `409 ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION`; hãy dùng endpoint deactivate. Response `200` là `DeletionResult` trong success envelope.

### `POST /products/bulk-price`

**Role:** TenantAdmin. Request:

```json
{
  "productIds": ["uuid"],
  "operation": "Increase",
  "mode": "Percentage",
  "value": 5.0,
  "expectedVersions": { "product-uuid": 4 }
}
```

`operation` nhận `Increase` hoặc `Decrease`; `mode` nhận `Percentage` hoặc `FixedAmount`; `value` là độ lớn dương, không gửi số âm để biểu diễn giảm giá. UI tự chọn cách hiển thị dấu. Giá sau điều chỉnh vẫn phải lớn hơn 0. Server giới hạn số lượng, lọc theo tenant, transaction toàn bộ và tăng `Version`.
Errors: `ERR_ACCESS_DENIED`, `ERR_REQUIRED_FIELD`, `ERR_UNKNOWN_FIELD`, `ERR_INVALID_ENUM`, `ERR_INVALID_DECIMAL`, `ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE`, `ERR_PRICE_MUST_BE_POSITIVE`, `ERR_VERSION_CONFLICT`.

### Categories

`GET /categories?search=&cursor=&pageSize=&includeInactive=` cho phép Staff và TenantAdmin đọc category active trong tenant; TenantAdmin có thể gửi `includeInactive=true`. Search theo tên/mã; sort `sortOrder ASC, name ASC, id ASC`; paging dùng cursor keyset opaque. `POST /categories` và `PATCH /categories/{id}` chỉ dành cho TenantAdmin. Code duy nhất trong tenant; category cha phải cùng tenant và không thể tạo chu trình. PATCH nhận `name`, `parentId`, `sortOrder`, `expectedVersion`; version cũ trả `409 ERR_VERSION_CONFLICT`.
Errors: `ERR_ACCESS_DENIED`, `ERR_CATEGORY_NOT_FOUND`, `ERR_DUPLICATE_CATEGORY_CODE`, `ERR_CATEGORY_PARENT_CYCLE`, `ERR_VERSION_CONFLICT`.

### TenantAdmin quản lý tài khoản Staff

Các endpoint sau chỉ dành cho TenantAdmin và chỉ tác động đến tài khoản role `Staff` thuộc tenant trong JWT. Không nhận `tenantId` hoặc `role` từ request để xác định quyền.

#### `GET /tenant/staff?search=&cursor=&pageSize=`

Liệt kê Staff của tenant, bao gồm tài khoản inactive; search theo username/display name, sort `displayName ASC, id ASC`, dùng cursor keyset và không chứa credential.

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

### Customers

TenantAdmin có CRUD customer trong tenant theo các giới hạn nghiệp vụ. Staff được tìm kiếm/đọc và tạo customer phục vụ lập invoice; Staff không được sửa hoặc xóa customer. Mọi thao tác lấy tenant từ JWT.

#### `GET /customers?search=&cursor=&pageSize=&updatedSince=&includeInactive=`

**Role:** Staff, TenantAdmin. Staff chỉ nhận customer active; gửi `includeInactive=true` với role Staff trả `403 ERR_ACCESS_DENIED`. TenantAdmin có thể đặt `includeInactive=true` để xem customer đã deactivate. Chỉ trả customer thuộc tenant hiện tại. `search` tìm theo tên, mã customer hoặc số điện thoại; sort `fullName ASC, id ASC`. Mobile mới dùng cursor keyset opaque: request đầu gửi `cursor=` rỗng, các trang sau truyền nguyên `nextCursor`. `page` chỉ được hỗ trợ như legacy mode cho client API v1 cũ.

#### `GET /customers/{id}`

**Role:** Staff, TenantAdmin. Trả chi tiết customer trong tenant hiện tại theo `Customer` schema. Staff chỉ đọc customer active; customer inactive hoặc khác tenant trả `404 ERR_CUSTOMER_NOT_FOUND`. TenantAdmin đọc được customer active/inactive để quản lý.

#### `POST /customers`

**Role:** Staff, TenantAdmin. Request gồm `code`, `fullName`, và các field tùy chọn `phone`, `email`, `address`. `code` duy nhất trong tenant.
Header `Idempotency-Key` là bắt buộc. API trả `202` với operation id sau publisher confirm; worker tạo customer và phát snapshot realtime sau khi commit.
Errors: `ERR_ACCESS_DENIED`, `ERR_INVALID_REQUEST`, `ERR_DUPLICATE_CUSTOMER_CODE`.

#### `PATCH /customers/{id}`

**Role:** TenantAdmin. Chỉ cập nhật `fullName`, `phone`, `email`, `address`; bắt buộc `expectedVersion`. Không nhận `tenantId`, `isActive` hoặc version do client tự gán. Version cũ trả `409 ERR_VERSION_CONFLICT`.
Errors: `ERR_ACCESS_DENIED`, `ERR_CUSTOMER_NOT_FOUND`, `ERR_VERSION_CONFLICT`.

#### `POST /customers/{id}/deactivate`

**Role:** TenantAdmin. Request `{ "expectedVersion": 3 }`. Đặt customer thành inactive, giữ nguyên lịch sử và FK từ invoice; customer inactive không được chọn cho invoice mới. Lặp deactivate trên customer inactive không tạo side effect khác.
Errors: `ERR_CUSTOMER_NOT_FOUND`, `ERR_VERSION_CONFLICT`.

#### `DELETE /customers/{id}?expectedVersion=`

**Role:** TenantAdmin. Chỉ hard-delete customer chưa được tham chiếu bởi invoice hoặc dữ liệu lịch sử. Nếu customer đã được sử dụng, trả `409 ERR_CUSTOMER_DELETE_REQUIRES_DEACTIVATION`; dùng deactivate để giữ nguyên tham chiếu/audit. Response `200` trả `DeletionResult` trong success envelope.
Errors: `ERR_CUSTOMER_NOT_FOUND`, `ERR_CUSTOMER_DELETE_REQUIRES_DEACTIVATION`, `ERR_VERSION_CONFLICT`.

Nếu feature `RequireCustomerPhone` bật, invoice gắn customer bắt buộc có phone; kiểm tra ở server khi sync.

### `GET /tenant/profile`

**Role:** TenantAdmin. Trả thông tin tenant gắn với access token hiện tại: `id`, `code`, `name`, `status`, `timeZone`, `currencyCode`, `version`, `createdAtUtc`, `updatedAtUtc`. Không nhận `tenantId` từ path, query hoặc body; server lấy tenant từ JWT context. Endpoint này chỉ đọc tenant hiện tại, không cấp quyền quản trị tenant khác như `/platform/tenants/{tenantId}`.

Errors: `ERR_UNAUTHENTICATED`, `ERR_TENANT_NOT_FOUND`.

## 4. Invoice

### Invoice business status và local sync status

Wire response giữ field `status`, có kiểu enum `InvoiceStatus`:

| `InvoiceStatus` | Ý nghĩa |
|---|---|
| `Active` | Invoice đã được server ghi nhận và chưa thuộc cặp bút toán đảo |
| `Reversed` | Invoice thuộc cặp bút toán đảo; áp dụng cho invoice gốc và invoice Reverse theo reverse-entry rule |

`invoiceType` vẫn phân biệt `Sale`, `Reverse`, `Correction`; status không thể thay thế type. `SyncStatus` là enum local của mobile, không phải trạng thái server và không được đọc từ `Invoice.status`:

| `SyncStatus` | Ý nghĩa |
|---|---|
| `Draft` | Hóa đơn chỉ được lưu local, chưa chốt để sync |
| `Pending` | Đã chốt local, chờ gửi |
| `Sending` | Client đang gửi |
| `Queued` | API đã nhận command, server chưa xác nhận commit |
| `Synced` | Server đã commit, xác nhận bằng event/pull |
| `Failed` | Bị từ chối hoặc cần xử lý; lưu `LastErrorCode` |

Trạng thái mạng `Online`/`Offline` và hoạt động pull `Downloading` là trạng thái app-wide riêng, không thuộc `InvoiceStatus` hay per-invoice `SyncStatus`. API không trả `SyncStatus`.

### `POST /invoices/sync`

**Role:** Staff, TenantAdmin. Đây là command idempotent và là endpoint chính cho offline outbox.
Header `Idempotency-Key` là bắt buộc và phải trùng `clientOperationId`; `payloadVersion` hiện hỗ trợ giá trị `1`.

Sale `quantity` và `unitPrice` phải lớn hơn 0. `taxRate`/`taxAmount` phải bằng 0 trong v1; không áp dụng nghiệp vụ tính thuế cho đến phần mở rộng sau. Hóa đơn Reverse và Correction do endpoint tương ứng tạo ở server, không được gửi như Sale payload từ client.
Money và quantity dùng `decimal` tối đa 4 chữ số thập phân. Server kiểm tra công thức bằng decimal chính xác, không tự làm tròn hoặc áp tolerance; client gửi `roundingAmount` tường minh nếu có. Chưa áp dụng `MidpointRounding` hay quy tắc làm tròn theo currency cho đến khi quy tắc nghiệp vụ được định nghĩa.

Request:

```json
{
  "clientOperationId": "uuid",
  "payloadVersion": 1,
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

Response `202` sau khi RabbitMQ xác nhận đã nhận lệnh:

```json
{
  "data": {
    "operationId": "uuid",
    "status": "Queued"
  },
  "meta": { "correlationId": "uuid", "serverTimeUtc": "2026-10-08T09:00:00Z" }
}
```

Worker xác thực invoice và ghi invoice, `sync_operations`, `server_changes`, outbox trong cùng transaction. Sau commit, client nhận `DataCreated` qua SignalR với invoice snapshot và `serverSequence`; lệnh lỗi phát `CommandRejected` tới user tạo lệnh. Retry cùng operation id và payload không tạo invoice mới; payload khác được reject.

Errors: `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_PRODUCT_NOT_FOUND`, `ERR_PRODUCT_INACTIVE`, `ERR_CUSTOMER_NOT_FOUND`, `ERR_CUSTOMER_INACTIVE`, `ERR_INVOICE_TOTAL_INVALID`, `ERR_CUSTOMER_PHONE_REQUIRED`, `ERR_SYNC_CONFLICT`.

### `GET /invoices?search=&from=&to=&invoiceType=&cursor=&pageSize=`

**Role:** Staff, TenantAdmin. Staff chỉ thấy invoice do chính mình tạo (`createdByUserId` = user trong JWT); TenantAdmin thấy toàn bộ invoice trong tenant. Tenant và user lấy từ JWT, không nhận từ query.
`search` tìm invoice number theo substring hoặc invoice ID/barcode theo exact match. `from` (bao gồm) và `to` (không bao gồm) lọc theo `clientCreatedAtUtc`; `invoiceType` nhận `Sale`, `Reverse` hoặc `Correction`. Kết quả sắp xếp `clientCreatedAtUtc DESC, id DESC`. Mobile mới dùng cursor keyset opaque: request đầu gửi `cursor=` rỗng, các trang sau truyền nguyên `nextCursor`. `page` chỉ được hỗ trợ như legacy mode cho client API v1 cũ. `pageSize` tối đa 200, mặc định 50. Response cursor gồm `nextCursor`, `pageSize`, `hasMore`; cursor hết hạn sau 15 phút. Dùng `GET /invoices/{id}` để lấy chi tiết.
Errors: `ERR_INVALID_PAGING`, `ERR_INVALID_ENUM`.

### `GET /invoices/{id}`

**Role:** Staff, TenantAdmin. Chỉ đọc invoice cùng tenant, gồm detail snapshot và thông tin reverse. Staff chỉ đọc invoice do chính mình tạo; invoice của người khác trả `404 ERR_INVOICE_NOT_FOUND`.

Staff và TenantAdmin được in invoice đã đọc trên thiết bị client. App dựng bản in từ invoice snapshot và `ReceiptTemplate` trong `/settings/effective`, rồi gửi tới máy in local/Bluetooth; in không thay đổi invoice và không có server-side print endpoint. Nếu in lỗi, mobile dùng `ERR_PRINT_FAILED`; invoice không rollback.

### `POST /invoices/{id}/reverse`

**Role:** Chỉ TenantAdmin. Staff không được reverse invoice, kể cả khi gửi thông tin/PIN của Admin. Actor được lấy từ JWT; không nhận `adminUserId` hoặc PIN từ request. Request:

```json
{ "reasonCode": "CUSTOMER_CANCELLED" }
```

Server lock invoice gốc, kiểm tra chưa reverse, tạo invoice âm mới và outbox event trong cùng transaction. Không update/delete detail gốc.
`Idempotency-Key` là UUID bắt buộc và `X-Device-Id` phải được gửi; retry cùng key/payload trả lại kết quả đã áp dụng, payload khác trả `409 ERR_SYNC_PAYLOAD_MISMATCH`. Response thành công là `201`.

Errors: `ERR_ACCESS_DENIED`, `ERR_INVOICE_NOT_FOUND`, `ERR_INVOICE_ALREADY_REVERSED`, `ERR_INVOICE_IMMUTABLE`.

### `POST /invoices/{id}/correction`

**Role:** TenantAdmin. Thực hiện reverse invoice cũ rồi tạo invoice correction mới; request chứa detail mới, lý do và `roundingAmount` tùy chọn (mặc định `0`). Không mutate invoice cũ. `Idempotency-Key` UUID và `X-Device-Id` là bắt buộc. Server tính lại subtotal/discount từ details, yêu cầu tax bằng 0, tính `totalAmount = subtotal - discountAmount + roundingAmount` và yêu cầu total lớn hơn 0; reverse, correction, audit, sync change và outbox commit atomically. Retry cùng key/payload trả lại kết quả correction đã tạo.

Request:

```json
{
  "reasonCode": "PRICE_CORRECTION",
  "roundingAmount": 0,
  "details": [{
    "lineNumber": 1,
    "productId": "uuid",
    "skuSnapshot": "SKU-001",
    "productNameSnapshot": "Coffee",
    "unitSnapshot": "cup",
    "quantity": 1,
    "unitPrice": 100000,
    "discountAmount": 0,
    "taxRate": 0,
    "taxAmount": 0,
    "lineTotal": 100000
  }]
}
```

`details` phải có tối thiểu một dòng theo schema `SaleInvoiceDetailRequest`. `roundingAmount` là decimal có thể âm hoặc dương; client không gửi chuỗi tiền đã format.

## 5. Sync pull và feature flags

### `GET /sync/pull?afterSequence=&limit=`

**Role:** Staff, TenantAdmin. Trả changes của tenant hiện tại có `serverSequence > afterSequence`, theo thứ tự tăng dần. `afterSequence` mặc định `0`; `limit` mặc định `50`, tối đa `200`. Sequence được ghi cùng transaction tạo entity và không dựa vào đồng hồ client.

Response dùng `data` là danh sách change; `meta.paging` có `nextSequence`, `hasMore`, `limit`. Nếu không có change mới, `nextSequence` giữ nguyên `afterSequence`.

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

#### `GET /platform/tenants?search=&cursor=&pageSize=&includeDeleted=`

Liệt kê tenant, mặc định không gồm tenant đã soft-delete. `includeDeleted=true` chỉ dành cho SuperAdmin. Search theo code/tên, sort `name ASC, id ASC`; response dùng cursor keyset.

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

#### `GET /platform/tenants/{tenantId}/admins?search=&cursor=&pageSize=`

Liệt kê tài khoản TenantAdmin của tenant, gồm cả tài khoản đã vô hiệu hóa. Search theo username/display name/email, sort `displayName ASC, id ASC`; response dùng collection envelope và cursor keyset; không trả password, password hash, PIN hoặc token.

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

Validation codes: `ERR_INVALID_REQUEST`, `ERR_REQUIRED_FIELD`, `ERR_UNKNOWN_FIELD`, `ERR_INVALID_FIELD`, `ERR_INVALID_UUID`, `ERR_INVALID_EMAIL`, `ERR_INVALID_TIME_ZONE`, `ERR_INVALID_PAGING`, `ERR_INVALID_ENUM`, `ERR_INVALID_DECIMAL`, `ERR_PRICE_MUST_BE_POSITIVE`, `ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE`, `ERR_INVALID_QUANTITY`.

`ERR_ACCESS_DENIED`, `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_INVALID_CREDENTIALS`, `ERR_PRODUCT_NOT_FOUND`, `ERR_PRODUCT_INACTIVE`, `ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION`, `ERR_DUPLICATE_SKU`, `ERR_DUPLICATE_BARCODE`, `ERR_CUSTOMER_NOT_FOUND`, `ERR_CUSTOMER_INACTIVE`, `ERR_CUSTOMER_PHONE_REQUIRED`, `ERR_CUSTOMER_DELETE_REQUIRES_DEACTIVATION`, `ERR_DUPLICATE_CUSTOMER_CODE`, `ERR_INVOICE_NOT_FOUND`, `ERR_INVOICE_IMMUTABLE`, `ERR_INVOICE_ALREADY_REVERSED`, `ERR_INVOICE_TOTAL_INVALID`, `ERR_SYNC_DUPLICATE`, `ERR_SYNC_PAYLOAD_MISMATCH`, `ERR_SYNC_CONFLICT`, `ERR_STAFF_NOT_FOUND`, `ERR_STAFF_DELETE_REQUIRES_DEACTIVATION`, `ERR_DUPLICATE_USERNAME`, `ERR_VERSION_CONFLICT`, `ERR_SETTING_NOT_FOUND`, `ERR_FORCE_UPDATE_REQUIRED`.

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
- Backend reject mọi JSON property không khai báo trong request schema, kể cả object lồng nhau; trả `400 ERR_UNKNOWN_FIELD` và `fieldErrors` chứa đường dẫn property. Không ignore field lạ và không mass-assign.
- Field bắt buộc bị thiếu, `null` hoặc rỗng trả `400 ERR_REQUIRED_FIELD`. Sai type/format/constraint trả error code cụ thể (`ERR_INVALID_FIELD`, `ERR_INVALID_UUID`, `ERR_INVALID_EMAIL`, `ERR_INVALID_TIME_ZONE`, `ERR_INVALID_ENUM`, `ERR_INVALID_DECIMAL` hoặc `ERR_INVALID_PAGING` theo trường hợp). JSON không parse được trả `ERR_INVALID_REQUEST`.
- Giá sản phẩm/unit price và độ lớn bulk adjustment phải dương; lỗi dùng `ERR_PRICE_MUST_BE_POSITIVE` hoặc `ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE`. Chiều tăng/giảm được truyền riêng, không mã hóa bằng dấu âm.
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

Mobile mới dùng cursor keyset opaque cho mọi list có paging, ngoại trừ `GET /sync/pull` dùng `afterSequence`/`nextSequence` để phục hồi thay đổi tuần tự. API v1 tạm hỗ trợ `page` offset deprecated để giữ tương thích client cũ. Nếu có query key `cursor`, kể cả giá trị rỗng, request chọn cursor mode; nếu không có `cursor`, request chọn legacy page mode (`page` mặc định là 1). Không gửi đồng thời `page` và `cursor`; kết hợp cả hai trả `400 ERR_INVALID_PAGING`. Cursor list được ràng buộc với tenant/user scope, từ khóa đã chuẩn hóa, filter và sort; client không tự tạo, giải mã, sửa hoặc dùng lại cursor cho query khác. Server dùng thứ tự xác định và khóa phụ ID duy nhất. Cursor hết hạn sau 15 phút; cursor không hợp lệ/hết hạn trả `400 ERR_INVALID_PAGING`; client bỏ cursor và tải lại bằng `cursor=`.

Với các query search, trim và chuẩn hóa Unicode NFC, không phân biệt hoa thường; query rỗng sau chuẩn hóa nghĩa là không filter, text fields dùng substring match và identifiers/barcodes dùng exact match khi endpoint quy định. Local và server phải dùng cùng fields/semantics. `pageSize` mặc định 50, tối đa 200. Cursor response có `nextCursor`, `hasMore`, `pageSize`; legacy page response giữ `page`, `hasMore`, `pageSize`. Không yêu cầu `totalCount`.

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

Query trong bảng là contract ưu tiên cho client mới. API v1 vẫn nhận `page` deprecated theo mục compatibility để hỗ trợ client cũ; không gửi `page` cùng `cursor`.

| Method | Endpoint | Request body/query | Response `data` | Status thành công |
|---|---|---|---|---:|
| POST | `/auth/login` | `LoginRequest` | `LoginResponse` | 200 |
| POST | `/auth/refresh` | `RefreshTokenRequest` | `TokenResponse` | 200 |
| POST | `/auth/logout` | Không body | `LogoutResult` | 200 |
| GET | `/products` | query `search,cursor,pageSize,updatedSince,includeInactive` | `Product[]` + cursor paging | 200 |
| POST | `/products` | `CreateProductRequest` | `Product` | 201 |
| PATCH | `/products/{id}` | `UpdateProductRequest` + `expectedVersion` | `Product` | 200 |
| POST | `/products/{id}/deactivate` | `expectedVersion` | `Product` | 200 |
| DELETE | `/products/{id}` | query `expectedVersion` | `DeletionResult` | 200 |
| POST | `/products/bulk-price` | `BulkPriceRequest` | `BulkPriceResult` | 200 |
| GET | `/tenant/staff` | query `search,cursor,pageSize` | `StaffAccount[]` + cursor paging | 200 |
| POST | `/tenant/staff` | `CreateStaffRequest` | `StaffAccount` | 201 |
| GET | `/tenant/staff/{userId}` | path `userId` | `StaffAccount` | 200 |
| PATCH | `/tenant/staff/{userId}` | `UpdateStaffRequest` + `expectedVersion` | `StaffAccount` | 200 |
| POST | `/tenant/staff/{userId}/deactivate` | Không body | `StaffAccount` | 200 |
| DELETE | `/tenant/staff/{userId}` | path `userId` | `DeletionResult` | 200 |
| GET | `/categories` | query `search,cursor,pageSize,includeInactive` | `Category[]` + cursor paging | 200 |
| POST | `/categories` | `CreateCategoryRequest` | `Category` | 201 |
| PATCH | `/categories/{id}` | `UpdateCategoryRequest` + version | `Category` | 200 |
| GET | `/customers` | query `search,cursor,pageSize,updatedSince,includeInactive` | `Customer[]` + cursor paging; Staff chỉ thấy active | 200 |
| POST | `/customers` | `CreateCustomerRequest` | `Customer` | 201 |
| PATCH | `/customers/{id}` | TenantAdmin; `UpdateCustomerRequest` + `expectedVersion` | `Customer` | 200 |
| POST | `/customers/{id}/deactivate` | TenantAdmin; `expectedVersion` | Inactive `Customer` | 200 |
| DELETE | `/customers/{id}` | TenantAdmin; query `expectedVersion` | `DeletionResult` | 200 |
| GET | `/invoices` | query `search,from,to,invoiceType,cursor,pageSize`; Staff chỉ thấy invoice của mình | `InvoiceSummary[]` + cursor paging | 200 |
| GET | `/invoices/{id}` | path `id` | `Invoice` | 200 |
| POST | `/invoices/sync` | `SyncInvoiceRequest` + idempotency | `OperationResult<Invoice>` | 201/200 |
| POST | `/invoices/{id}/reverse` | TenantAdmin; `ReverseInvoiceRequest` + idempotency | `OperationResult<Invoice>` | 201 |
| POST | `/invoices/{id}/correction` | `CorrectionInvoiceRequest` + idempotency | `OperationResult<Invoice>` | 201 |
| GET | `/sync/pull` | `afterSequence,limit` | `SyncChange[]` + sequence paging | 200 |
| GET | `/settings/effective` | Không body | `EffectiveSettings` | 200 |
| PUT | `/settings/{key}` | `UpdateSettingRequest` + version; TenantAdmin tenant scope | `Setting` | 200 |
| GET | `/platform/tenants` | query `search,cursor,pageSize,includeDeleted` | `PlatformTenant[]` + cursor paging | 200 |
| POST | `/platform/tenants` | `CreatePlatformTenantRequest` including initial `TenantAdmin` | `CreatePlatformTenantResponse` | 201 |
| GET | `/platform/tenants/{tenantId}` | path `tenantId`; optional `includeDeleted` | `PlatformTenant` | 200 |
| PATCH | `/platform/tenants/{tenantId}` | `UpdatePlatformTenantRequest` + `expectedVersion` | `PlatformTenant` | 200 |
| POST | `/platform/tenants/{tenantId}/suspend` | Không body | `PlatformTenant` | 200 |
| POST | `/platform/tenants/{tenantId}/reactivate` | Không body | `PlatformTenant` | 200 |
| DELETE | `/platform/tenants/{tenantId}` | path `tenantId` | `DeletePlatformTenantResponse` (soft-delete) | 200 |
| GET | `/platform/tenants/{tenantId}/admins` | query `search,cursor,pageSize` | `TenantAdminAccount[]` + cursor paging | 200 |
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

Response HTTP của sync là `202` khi command được xác nhận bởi broker:

```json
{
  "data": { "operationId": "uuid", "status": "Queued" },
  "meta": { "correlationId": "uuid", "serverTimeUtc": "2026-10-08T09:00:00Z" }
}
```

Retry cùng operation id và cùng payload được worker deduplicate. Client theo dõi kết quả bằng SignalR `DataCreated`/`CommandRejected`; `GET /sync/pull` khôi phục event thành công bị lỡ.

### 11.4. Pull sync

Response envelope:

```json
{
  "data": [
    {
      "serverSequence": 123,
      "changeType": "Upsert",
      "entityType": "Invoice",
      "entityId": "uuid",
      "entityVersion": 1,
      "operationId": "uuid",
      "occurredAtUtc": "2026-10-08T09:01:00Z",
      "entity": {}
    }
  ],
  "meta": {
    "correlationId": "uuid",
    "paging": { "nextSequence": 123, "hasMore": false, "limit": 50 }
  }
}
```

Client chỉ lưu `nextSequence` sau khi toàn bộ `data` đã upsert vào SQLite thành công. Nếu lỗi giữa batch, gọi lại `afterSequence` cũ là an toàn.

### SignalR realtime

Hub: `/hubs/pos`, xác thực bằng JWT. Server tự thêm connection vào group `tenant:{TenantId}`; không nhận group/tenant từ client. Event `DataCreated` chứa `messageId`, `serverSequence`, `entityType`, `entityId`, `entityVersion`, `occurredAtUtc`, và full `entity` snapshot. Client deduplicate sequence/message id rồi chèn snapshot lên đầu danh sách phù hợp mà không gọi refresh API. Event `CommandRejected` chỉ gửi tới user khởi tạo và có `operationId`, `errorCode`.

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
