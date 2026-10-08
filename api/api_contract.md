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
| `X-Device-Id` | Có với mobile | Định danh thiết bị đã đăng ký |
| `X-Correlation-Id` | Khuyến nghị | Truy vết request |
| `Idempotency-Key` | Có với command retry | Nên bằng `ClientOperationId` |

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

### Response lỗi đa ngôn ngữ

API không trả text nghiệp vụ hardcode. Mobile dùng `errorCode` để tra `vi.json`/`en.json`.

```json
{
  "error": {
    "code": "ERR_PRODUCT_NOT_FOUND",
    "args": { "productId": "uuid" },
    "fieldErrors": [],
    "retryable": false
  },
  "meta": { "correlationId": "uuid" }
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
  "password": "secret",
  "deviceId": "device-uuid",
  "platform": "Android",
  "appVersion": "2.1.0"
}
```

Response 200:

```json
{
  "data": {
    "accessToken": "jwt",
    "expiresAtUtc": "2026-10-08T11:00:00Z",
    "refreshToken": "opaque-token",
    "user": {
      "id": "uuid",
      "tenantId": "uuid",
      "role": "Staff",
      "displayName": "Cashier"
    },
    "device": { "id": "uuid", "isRevoked": false }
  }
}
```

Errors: `ERR_INVALID_CREDENTIALS`, `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_FORCE_UPDATE_REQUIRED`.

### `POST /auth/refresh`

Nhận refresh token, kiểm tra hash trong `user_devices`, trạng thái user/device và rotate token. Không lưu plaintext refresh token.

### `POST /auth/logout`

Thu hồi session hiện tại. Không xóa dữ liệu server.

## 3. Products, categories và customers

### `GET /products?search=&page=&pageSize=&updatedSince=`

**Role:** Staff, TenantAdmin.  
Trả danh sách sản phẩm active hoặc thay đổi sau `updatedSince` để pull incremental.

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
    "settingsVersion": 12
  }
}
```

### `PUT /settings/{key}`

**Role:** TenantAdmin với tenant setting; SuperAdmin với global setting. Giá trị phải validate theo schema của key.

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

`ERR_ACCESS_DENIED`, `ERR_TENANT_INACTIVE`, `ERR_DEVICE_REVOKED`, `ERR_INVALID_CREDENTIALS`, `ERR_PRODUCT_NOT_FOUND`, `ERR_PRODUCT_INACTIVE`, `ERR_DUPLICATE_SKU`, `ERR_DUPLICATE_BARCODE`, `ERR_CUSTOMER_PHONE_REQUIRED`, `ERR_INVOICE_NOT_FOUND`, `ERR_INVOICE_IMMUTABLE`, `ERR_INVOICE_ALREADY_REVERSED`, `ERR_INVOICE_TOTAL_INVALID`, `ERR_ADMIN_PIN_INVALID`, `ERR_SYNC_DUPLICATE`, `ERR_SYNC_PAYLOAD_MISMATCH`, `ERR_SYNC_CONFLICT`, `ERR_FORCE_UPDATE_REQUIRED`.

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
| GET | `/products` | query `search,page,pageSize,updatedSince,cursor` | `Product[]` + paging | 200 |
| POST | `/products` | `CreateProductRequest` | `Product` | 201 |
| PATCH | `/products/{id}` | `UpdateProductRequest` + `expectedVersion` | `Product` | 200 |
| POST | `/products/bulk-price` | `BulkPriceRequest` | `BulkPriceResult` | 200 |
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
| PUT | `/settings/{key}` | `UpdateSettingRequest` + version | `Setting` | 200 |
| GET | `/config/version` | Không body | `VersionConfig` | 200 |

Tất cả endpoint trong bảng đều trả `meta.correlationId`. Endpoint collection luôn trả `meta.paging`, kể cả khi `data` rỗng.

---

## 11. Format request/response chi tiết theo nhóm API

### 11.1. Login

Request `LoginRequest`:

```json
{
  "username": "cashier01",
  "password": "secret",
  "deviceId": "device-uuid",
  "platform": "Android",
  "appVersion": "2.1.0"
}
```

Response `LoginResponse`:

```json
{
  "accessToken": "jwt",
  "refreshToken": "opaque-token",
  "accessTokenExpiresAtUtc": "2026-10-08T11:00:00Z",
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
| `ERR_IDEMPOTENCY_KEY_MISMATCH` | Header và body operation id khác nhau |
| `ERR_IDEMPOTENCY_KEY_REUSED` | Một key dùng cho payload khác |
| `ERR_UNSUPPORTED_PAYLOAD_VERSION` | Server không hỗ trợ payload version |
| `ERR_INVALID_ENUM` | Giá trị enum không hợp lệ |
| `ERR_INVALID_DECIMAL` | Tiền/số lượng sai precision hoặc range |
| `ERR_VERSION_CONFLICT` | `expectedVersion` đã cũ |
| `ERR_CURSOR_EXPIRED` | Cursor pull không còn hợp lệ |

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
