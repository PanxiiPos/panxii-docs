# API CONTRACT - POS SaaS

**Version:** v1  
**Base URL:** `/api/v1`  
**Format:** JSON UTF-8  
**Authentication:** `Authorization: Bearer <JWT>`  
**Database:** MySQL 8 + EF Core 8

## 1. Quy ước chung

### Catalog offline commands (mobile)

- `POST /products/commands`: TenantAdmin, bắt buộc `Idempotency-Key` UUID khác rỗng; trả `202` với `{operationId,status:"Queued"}`. Worker kiểm tra lại tài khoản/tenant/quyền khi thực thi.
- Body: `{action,clientEntityId,productId,variantId?,expectedVersion,data,prices?}`. `clientEntityId` là UUID local khác rỗng; `data` luôn là object. Actions: `create`, `update`, `createVariant`, `updateVariant`, `deactivate`, `deactivateVariant`, `activate`, `activateVariant`, `delete`, `deleteVariant`, `prices`. `data` theo DTO của hành động legacy tương ứng; `expectedVersion` ngoài cùng là authoritative.
- `prices`: 1–100 `{productId,variantId?,expectedVersion,listedPrice,calculationQuantity}`. Cả hai số phải >0, tối đa 4 chữ số thập phân, `decimal(19,4)`. Server tính `basePrice = listedPrice × calculationQuantity`, làm tròn tới 4 chữ số thập phân (midpoint away from zero). Client gửi decimal invariant dạng chuỗi để giữ chính xác; không nhận cùng physical ID hai lần (product/default variant). Điều chỉnh tăng/giảm trên mobile áp dụng vào `listedPrice`, không sửa trực tiếp giá bán.
- `GET /products/commands/{operationId}`: TenantAdmin, chỉ tenant hiện tại và người tạo operation. `200 data={status:"Completed"|"Rejected",errorCode?,result?}`; chưa có receipt trả `404 ERR_OPERATION_NOT_FOUND` (có thể vẫn đang chờ trong RabbitMQ). Payload mismatch giữ nguyên lỗi `ERR_SYNC_PAYLOAD_MISMATCH`; không thay payload của UUID đã gửi.
- Result: `{clientEntityId,entityId,products:[],variants:[],deletedProductIds:[],deletedVariantIds:[]}`; `entityId` là ID server của resource vừa tạo, hoặc ID target/batch. `products`/`variants` là snapshot đầy đủ của các product bị tác động, không phải chỉ các dòng đổi giá. `deletedVariantIds` có thể null ở receipt cũ.
- Mutation, receipt chứa result, audit, `CatalogBatch` server change và realtime outbox cùng transaction. Một version sai từ chối toàn batch; retry UUID/payload không thực thi lại. API legacy giữ nguyên để tương thích.
- Không xóa riêng default variant; variant đã có lịch sử hóa đơn phải ngừng bán. Không xóa variant active cuối cùng của product active. Cho phép ngừng/bán lại riêng bất kỳ variant (kể cả default/cuối cùng) mà không đổi product gốc hoặc variant khác.
- `deactivate`/`activate` đổi trạng thái product và TẤT CẢ variant trong cùng transaction, kể cả variant đã ngừng riêng. `deactivateVariant`/`activateVariant` chỉ đổi variant chỉ định. Mỗi entity chỉ tăng version khi cờ active của chính nó đổi; invoice history không đổi. Variant chỉ bán được khi cả gốc và variant active.
- REST tương ứng: `POST /products/{id}/activate` và `POST /products/{productId}/variants/{variantId}/activate`, TenantAdmin only, body `{expectedVersion}`, trả envelope `Product`/`ProductVariant` giống deactivate; lỗi invalid request/not found/version conflict/access denied theo contract hiện có. Mobile tiếp tục dùng durable commands thay vì gọi REST trực tiếp.
- Product/variant response bổ sung `basePriceText`, `listedPriceText` và `calculationQuantityText` dạng invariant string; các field numeric giữ nguyên. Client mới ưu tiên string; không dùng floating point để tính giá.
- Variant update có thể gửi lại `extraData` legacy/schema chưa hỗ trợ **không đổi** (so sánh cấu trúc JSON với dữ liệu hiện tại) để sửa tên/giá mà không mất thuộc tính. Vẫn kiểm tra tenant và `expectedVersion`. Tạo mới hoặc thay đổi schema chưa hỗ trợ vẫn bị từ chối; không phải cơ chế ghi JSON tùy ý. Mobile dùng form tên/giá trị, tự tạo schema 1, không gửi tiền đã format theo locale.
- Pull `CatalogBatch.entity` dùng result shape trên; áp dụng snapshot/tombstone/ID mapping và sequence cùng local transaction. Client không xem `202` là sync thành công và không tự rebase conflict.
- Deployment: cập nhật API và worker cùng phiên bản. Worker cần `JWT_SECRET_KEY` hợp lệ (cùng cấu hình API) vì product service phụ thuộc cursor codec; thiếu key sẽ retry/DLQ trước khi ghi receipt. Local compose đã truyền biến này, không lưu giá trị secret vào source.

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

**Role:** TenantAdmin, Staff. Chỉ cập nhật hồ sơ của chính user từ JWT. Request bắt buộc có `username`, `displayName`, `email`, `phone`, `expectedVersion`; `email` và `phone` có thể là `null` để xóa giá trị. `username` là bất biến qua endpoint này: client phải gửi lại đúng username hiện tại; giá trị khác trả `400 ERR_INVALID_FIELD` với `args.field = "username"`. Version cũ trả `409 ERR_VERSION_CONFLICT`.

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

API hỗ trợ product gốc và các variant bán được. Để tương thích client cũ, product vẫn giữ SKU/đơn vị/giá legacy; khi tạo product, server đồng thời tạo variant mặc định có cùng ID với product. Client mới đọc danh sách variant và gửi `productVariantId` khi đồng bộ hóa đơn. `ProductNameSnapshot` là tên product gốc, còn `VariantNameSnapshot` là nhãn variant riêng. Không dùng `ExtraData` để thay thế SKU hoặc giá. Schema `ExtraData` của variant theo contract ở [database_design.md](../database/database_design.md).

### `GET /products?search=&cursor=&pageSize=&updatedSince=&includeInactive=`

**Role:** Staff, TenantAdmin.  
Staff chỉ nhận sản phẩm active. TenantAdmin có thể đặt `includeInactive=true` để xem cả sản phẩm đã vô hiệu hóa. `updatedSince` dùng để pull incremental. `search` tìm theo tên, SKU hoặc barcode. Kết quả sắp xếp `name ASC, id ASC`.
Mobile mới dùng cursor keyset opaque: request đầu gửi `cursor=` rỗng, các trang sau truyền nguyên `nextCursor`. `page` chỉ được hỗ trợ như legacy mode cho client API v1 cũ; không dùng trong client mới.

Màn catalog mobile dùng `GET /products/catalog?search=&cursor=&pageSize=&includeInactive=` thay vì list nhóm hàng. Endpoint mới trả `data: [{product: <Product>, variant: <ProductVariant>}]` cùng `meta.paging`; mỗi dòng là một biến thể bán được, bao gồm default variant đúng một lần. Search tìm tên gốc, tên biến thể, SKU, barcode và JSON đặc điểm đã lưu; không biến kết quả variant thành product gốc. Sort `variant.sku ASC, variant.id ASC`, `pageSize` 1–200, bắt buộc cursor mode (`cursor=` ở trang đầu); cursor gắn tenant/search/filter, có hạn 15 phút. Staff chỉ thấy cả parent và variant active; `includeInactive=true` chỉ TenantAdmin. Endpoint product legacy phía trên không đổi.

Cả `GET /products` và `GET /products/catalog` nhận query `categoryId=<uuid>` tùy chọn: lọc **đúng** category của product gốc, kết hợp với search/status/updatedSince hiện có, không tự gồm category con hoặc product chưa phân loại. Category không thuộc tenant hoặc không tồn tại trả collection rỗng; không rò rỉ ownership. Cursor gắn cả categoryId; không dùng cursor giữa hai category hoặc giữa có/không có filter (`400 ERR_INVALID_PAGING`). Bỏ categoryId vẫn giữ list legacy để tương thích API, không có nghĩa mobile có tab “Tất cả”. Product snapshot (list, catalog, detail và sync tạo mới) trả `categoryId` nullable và `extraData` nullable.

Mã hàng vẫn bắt buộc trong payload API và kiểm tra duy nhất trong tenant. Form mobile cho bỏ trống mã: trước enqueue tự tạo `MH-` + 12 ký tự theo [ADR-014](../architecture/adr/ADR-014-short-business-codes.md), giữ mã đó trong payload bền vững qua retry/remap; mã người dùng nhập không được thay thế. Không tạo mã mới khi replay operation. Sale mới dùng `INV-` + 12 ký tự cùng thuật toán ở app/backend; reverse/correction do backend tạo cũng dùng số ngắn. UUID identity không đổi; số hóa đơn đã lưu và nháp legacy dùng UUID đầy đủ được giữ nguyên. Deploy API/worker trước mobile để số ngắn không bị ghi lại thành số dài.

```json
{
  "data": [{
    "id": "uuid",
    "sku": "SKU-001",
    "barcode": "893...",
    "name": "Coffee",
    "unit": "cup",
    "basePrice": 25000.0000,
    "basePriceText": "25000",
    "listedPrice": 25000.0000,
    "listedPriceText": "25000",
    "calculationQuantity": 1.0000,
    "calculationQuantityText": "1",
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

**Role:** TenantAdmin. Request yêu cầu `sku`, `name`, `unit`, `basePrice`, `taxRate`; `barcode` và `categoryId` tùy chọn. Có thể gửi thêm `listedPrice` và `calculationQuantity` cùng nhau; khi có, `basePrice` phải bằng tích hai giá trị sau khi làm tròn 4 chữ số. Nếu không gửi, server giữ tương thích bằng cách đặt `listedPrice = basePrice`, `calculationQuantity = 1`. Barcode tối đa 64 ký tự và duy nhất trong tenant nếu có. Giá phải lớn hơn 0; `taxRate` giữ bằng 0 trong v1.

`extraData` tùy chọn là object JSON (không phải chuỗi): `{"schemaVersion":1,"attributes":{...}}`, tối đa **16 KiB UTF-8**. Schema/giới hạn giống tạo/cập nhật variant. Worker lưu cùng attributes vào product và default variant; áp dụng cả catalog command `action: "create"`. Bỏ/null extraData không thêm property vào queued payload cũ, không thay cách tính payload hash và không tái serialize command đã xếp hàng để xác minh hash.

Nếu gửi categoryId, API kiểm tra category cùng tenant và active trước khi queue; worker kiểm tra lại trước ghi (kể cả catalog create). Không tồn tại/khác tenant: `ERR_CATEGORY_NOT_FOUND`; inactive: `ERR_CATEGORY_INACTIVE`. Command đã nhận `202` vẫn có thể bị worker từ chối nếu category đổi trạng thái trước commit. categoryId vẫn tùy chọn ở API để tương thích command/client cũ; **mobile tạo mới bắt buộc chọn một category active**, không tạo product uncategorized. Dữ liệu sample/legacy chưa phân loại vẫn hợp lệ nhưng không xuất hiện trong category filter.
Header `Idempotency-Key` là bắt buộc. API publish command bền vững tới RabbitMQ rồi trả `202` với `{ "operationId": "uuid", "status": "Queued" }`. Worker mới tạo product; client nhận event `DataCreated` sau khi MySQL commit.

Errors: `ERR_ACCESS_DENIED`, `ERR_REQUIRED_FIELD`, `ERR_UNKNOWN_FIELD`, `ERR_INVALID_FIELD`, `ERR_INVALID_DECIMAL`, `ERR_PRICE_MUST_BE_POSITIVE`, `ERR_DUPLICATE_SKU`, `ERR_DUPLICATE_BARCODE`.

### `GET /products/{id}`

**Role:** Staff, TenantAdmin. Trả chi tiết product trong tenant hiện tại theo cùng `Product` schema như list. Staff chỉ đọc product active; product inactive hoặc khác tenant trả `404 ERR_PRODUCT_NOT_FOUND`. TenantAdmin đọc được product active/inactive để quản lý.

### `GET /products/{productId}/variants?includeInactive=`

**Role:** Staff, TenantAdmin. Trả các variant của product; mặc định chỉ variant active. `includeInactive=true` chỉ dành cho TenantAdmin. Response gồm `id`, `productId`, `sku`, `barcode`, `name`, `unit`, `basePrice`, `listedPrice`, `calculationQuantity`, `taxRate`, `isActive`, `version`, `extraData` và `updatedAtUtc`.

### `GET /products/{productId}/variants/{variantId}`

**Role:** Staff, TenantAdmin. Trả variant nếu thuộc product và tenant hiện tại. Staff chỉ đọc variant active; TenantAdmin có thể đọc variant inactive.

### `POST /products/{productId}/variants`

**Role:** TenantAdmin. Tạo variant bán được với SKU, unit, `basePrice`, `listedPrice` và `calculationQuantity`, cùng tùy chọn barcode, tên, `extraData`. `basePrice` được tính từ giá niêm yết nhân hệ số cho một đơn vị bán; nếu hai field mới bị bỏ qua thì dùng `basePrice` và hệ số 1 để tương thích client cũ. Barcode/SKU phải không trùng trong tenant ở kiểm tra nghiệp vụ. `taxRate` hiện phải bằng 0. Thành công trả `201` và Location tới variant mới.

`extraData` nhận object dạng `{"schemaVersion":1,"attributes":{...}}`; giới hạn JSON 16 KiB. Dùng attributes cho đặc tính hàng hóa linh hoạt như thương hiệu, độ dày, kích thước hoặc quy cách; dữ liệu định danh/bán hàng như SKU, unit, giá phải nằm ở field riêng.

### `PATCH /products/{productId}/variants/{variantId}`

**Role:** TenantAdmin. Cập nhật SKU/barcode/tên/unit/giá niêm yết/hệ số quy đổi/thuộc tính với `expectedVersion` bắt buộc. Giá bán được server tính lại; tích không khớp `basePrice` trả `ERR_INVALID_REQUEST`. Trả `409 ERR_VERSION_CONFLICT` nếu version cũ.

### `POST /products/{productId}/variants/{variantId}/deactivate`

**Role:** TenantAdmin. Vô hiệu hóa riêng variant bằng `expectedVersion`, kể cả default/variant active cuối cùng; không đổi product gốc hoặc variant khác. Variant đã xuất hiện trên hóa đơn không được xóa vật lý.

### `POST /products/{productId}/variants/{variantId}/activate`

**Role:** TenantAdmin. Body `{expectedVersion}`; bán lại riêng variant, trả `ProductVariant`. Không đổi gốc hoặc variant khác; bán thực tế vẫn yêu cầu gốc active. Version cũ trả `409 ERR_VERSION_CONFLICT`.

### `PATCH /products/{id}`

**Role:** TenantAdmin. Cho cập nhật `name`, `barcode`, `unit`, `basePrice`, `listedPrice`, `calculationQuantity`, `taxRate`, `categoryId` và bắt buộc `expectedVersion`. Giá bán được tính lại từ giá niêm yết × hệ số quy đổi; client cũ chỉ gửi `basePrice` được xử lý tương thích với hệ số 1. `barcode: null` dùng để xóa barcode; barcode khác null phải duy nhất trong tenant. Nếu version không khớp trả `409 ERR_VERSION_CONFLICT`; barcode trùng trả `409 ERR_DUPLICATE_BARCODE`. Staff không được sửa giá.

### `POST /products/{id}/deactivate`

**Role:** TenantAdmin. Vô hiệu hóa sản phẩm và TẤT CẢ biến thể trong cùng transaction, giữ nguyên dữ liệu lịch sử; không cho invoice mới sử dụng sản phẩm inactive. Request có `expectedVersion`; trả Product đã cập nhật.

### `POST /products/{id}/activate`

**Role:** TenantAdmin. Body `{expectedVersion}`; bán lại gốc và TẤT CẢ biến thể trong cùng transaction, kể cả biến thể đã ngừng riêng. Trả `Product`; không đổi invoice history. Version cũ trả `409 ERR_VERSION_CONFLICT`.

### `DELETE /products/{id}?expectedVersion=`

**Role:** TenantAdmin. Chỉ hard-delete sản phẩm chưa được tham chiếu bởi invoice hoặc dữ liệu lịch sử. Nếu đã được sử dụng, trả `409 ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION`; hãy dùng endpoint deactivate. Response `200` là `DeletionResult` trong success envelope.

### `POST /products/bulk-price`

**Role:** TenantAdmin. Request:

```json
{
  "productIds": ["uuid"],
  "variantIds": ["variant-uuid"],
  "operation": "Increase",
  "mode": "Percentage",
  "value": 5.0,
  "expectedVersions": { "product-uuid": 4 },
  "variantExpectedVersions": { "variant-uuid": 2 }
}
```

Chọn `productIds`, `variantIds`, hoặc cả hai; mỗi collection tối đa 100 ID và tổng target tối đa 100. `expectedVersions` phải khớp chính xác các product được chọn; `variantExpectedVersions` phải khớp chính xác các variant được chọn. Collection không chọn có thể bỏ qua cùng version map tương ứng.

Product ID nhắm vào giá gốc/legacy của product và đồng bộ giá default variant. Variant ID nhắm vào giá riêng của variant. Có thể chọn cả giá gốc lẫn variant trong cùng request; server tính riêng từng giá và cập nhật tất cả trong một transaction, trả `products` và `variants` đã thay đổi. Nếu product và default variant cùng được chọn, đó là một giá dùng chung nên chỉ áp dụng điều chỉnh một lần; không cộng dồn hai lần. Chọn variant khác thì giá variant đó được thay đổi độc lập.

`operation` nhận `Increase` hoặc `Decrease`; `mode` nhận `Percentage` hoặc `FixedAmount`; `value` là độ lớn dương, không gửi số âm để biểu diễn giảm giá. UI tự chọn cách hiển thị dấu. Giá sau điều chỉnh vẫn phải lớn hơn 0. Server lọc theo tenant, kiểm tra concurrency cho từng loại target và transaction toàn bộ.
Errors: `ERR_ACCESS_DENIED`, `ERR_REQUIRED_FIELD`, `ERR_UNKNOWN_FIELD`, `ERR_INVALID_REQUEST`, `ERR_INVALID_ENUM`, `ERR_INVALID_DECIMAL`, `ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE`, `ERR_PRICE_MUST_BE_POSITIVE`, `ERR_PRODUCT_NOT_FOUND`, `ERR_PRODUCT_VARIANT_NOT_FOUND`, `ERR_VERSION_CONFLICT`.

### Categories

`GET /categories?search=&cursor=&pageSize=&includeInactive=` cho phép Staff và TenantAdmin đọc category active trong tenant; TenantAdmin có thể gửi `includeInactive=true`. Search theo tên/mã; sort `sortOrder ASC, name ASC, id ASC`; paging dùng cursor keyset opaque. `POST /categories` và `PATCH /categories/{id}` chỉ dành cho TenantAdmin. Code duy nhất trong tenant; category cha phải cùng tenant và không thể tạo chu trình. PATCH nhận `name`, `parentId`, `sortOrder`, `expectedVersion`; version cũ trả `409 ERR_VERSION_CONFLICT`.

Mọi Category response gồm `id`, `parentId`, `code`, `name`, `isActive`, `sortOrder`, `version`, `updatedAtUtc`, **`productCount`**. Count là số product gốc tham chiếu trực tiếp category trong tenant, gồm cả product inactive; không đếm variant hoặc descendants.

Lifecycle chỉ dành cho TenantAdmin, scoped tenant từ JWT và ghi audit cùng transaction:

- `PATCH /categories/{id}/status` body `{"isActive":false,"expectedVersion":3}` (hoặc true để bật lại); thành công `200` theo envelope `{data: <Category>, meta: ...}`. Không cascade trạng thái xuống product/category con; dữ liệu tham chiếu được giữ.
- `DELETE /categories/{id}?expectedVersion=3`: hard delete chỉ khi **không có product nào** tham chiếu, kể cả product inactive, và không có category con. Thành công `204` không body. Có product: `409 ERR_CATEGORY_IN_USE` (chỉ bật/tắt, không xóa); có con: `409 ERR_CATEGORY_HAS_CHILDREN` (xử lý/chuyển category con trước, không cascade/detach ngầm).
- Version bắt buộc >= 1; version cũ trả `409 ERR_VERSION_CONFLICT`; không tồn tại/khác tenant `404 ERR_CATEGORY_NOT_FOUND`. Tạo/gán product hoặc category con tham gia concurrency token của category được tham chiếu, nên version có thể tăng khi số tham chiếu thay đổi. Nếu association/status/delete chạy đồng thời, một write bị conflict và rollback; client reload trước retry. Không tự retry request phá optimistic concurrency.

Mobile chỉ dùng category active làm tab, không tạo tab “Tất cả”. Khi lần đầu vào mà không có category active, chuyển quản lý category; tạo product yêu cầu category đang chọn. Trạng thái lỗi/offline không được suy diễn là danh sách category rỗng đã xác nhận.
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

**Role:** TenantAdmin, Staff. Trả thông tin tenant gắn với access token hiện tại: `id`, `code`, `name`, `status`, `timeZone`, `currencyCode`, `version`, `createdAtUtc`, `updatedAtUtc`. Không nhận `tenantId` từ path, query hoặc body; server lấy tenant từ JWT context. Endpoint này chỉ đọc tenant hiện tại, không cấp quyền quản trị tenant khác như `/platform/tenants/{tenantId}`.

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
Header `Idempotency-Key` là bắt buộc và phải trùng `clientOperationId`; `payloadVersion` hỗ trợ `1` (không phí dịch vụ) và `2` (có snapshot phí dịch vụ).

Sale `quantity` và `unitPrice` phải lớn hơn 0. `taxRate`/`taxAmount` phải bằng 0 trong v1; không áp dụng nghiệp vụ tính thuế cho đến phần mở rộng sau. Hóa đơn Reverse và Correction do endpoint tương ứng tạo ở server, không được gửi như Sale payload từ client.

`variantNameSnapshot` là tùy chọn: biến thể không có tên riêng gửi `null`. Trong kiểm tra catalog, `null` và `""` được coi cùng nghĩa không có tên; không thay bằng tên product và không bỏ kiểm tra ordinal khi tên biến thể thật tồn tại.
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
      "productVariantId": "uuid",
      "skuSnapshot": "SKU-001",
      "productNameSnapshot": "Coffee",
      "variantNameSnapshot": "Cup 500ml",
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

`productVariantId` và `variantNameSnapshot` có thể bỏ khi gửi từ client cũ; server phân giải về variant mặc định. Client mới phải gửi ID variant được chọn, SKU, tên product gốc, nhãn variant và unit khớp catalog hiện tại. Response hóa đơn lưu `productVariantId`, `variantNameSnapshot` và `variantExtraDataSnapshot` để giữ nguyên thông tin bán tại thời điểm tạo hóa đơn.

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

Staff và TenantAdmin được in invoice có quyền đọc, không phải nháp, khi `EnableWifiPrint=true` đã được xác nhận. App dựng A4/A5 từ invoice snapshot, `StoreProfile` và `ReceiptTemplate`, gửi PNG cùng `paper: A4/A5` qua HTTPS pin chứng chỉ tới bridge Windows trong LAN; bridge dùng driver Canon LBP 2900 theo đúng khổ. Không có backend print endpoint hay transport Bluetooth. Lỗi in không rollback invoice. Retry cùng tài liệu dùng UUID đã lưu; `submitted` chỉ xác nhận Windows queue, `uncertain` không tự in lại. Chi tiết [bridge](../../tools/windows-print-bridge/README.md).

### `GET /invoices/operations/{operationId}`

**Role:** tenant user; receipt chỉ được đọc trong tenant và actor phù hợp. Trả envelope với `data.status` (`Completed`, `Rejected`, hoặc `Queued` nếu đã có receipt đang xử lý), `data.result` là Invoice khi Completed, `data.errorCode` khi Rejected. Không tìm thấy trả 404; client giữ nguyên operation và kiểm tra lại hoặc resend nguyên payload, không đổi UUID.

### Phí dịch vụ invoice payloadVersion 2

V1 tiếp tục không phí. V2 thêm vào invoice:

```json
{
  "serviceCharges": [
    { "serviceId": "delivery", "labelSnapshot": "Giao hàng", "amount": 5000 }
  ],
  "serviceTotal": 5000
}
```

Tối đa 10 ID duy nhất, ID tối đa 80 ký tự, nhãn tối đa 120, amount không âm/decimal(19,4). `serviceTotal` phải bằng tổng snapshot amounts, không suy diễn từ tổng invoice hoặc tự sửa số tiền client. Sale mới đối chiếu định nghĩa Service trong ReceiptTemplate của tenant; dữ liệu cấu hình stale không được âm thầm đổi nhãn. Total = subtotal − discountAmount + taxAmount (0) + roundingAmount + serviceTotal. Giữ snapshot dù mẫu đổi sau bán.

Correction nhận thêm `payloadVersion` (mặc định 1), serviceCharges/serviceTotal. V1 correction bị từ chối nếu hóa đơn gốc có bất kỳ snapshot phí nào, kể cả phí 0; không âm thầm xóa phí. V2 giữ ID/nhãn gốc dù mẫu đổi, dịch vụ mới phải khớp định nghĩa đang bật. Reverse negate phí gốc và giữ nhãn/ID mà không lookup mẫu mới. Response invoice, realtime/pull và operation result đều mang snapshots; client cũ thiếu field xem như không có phí lịch sử, không tự tính lại số tiền server.

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
    "EnableWifiPrint": false,
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
    "settingsVersion": 12,
    "settingVersions": {
      "StoreProfile": 2,
      "ReceiptTemplate": 0,
      "ProductDisplay": 3,
      "AutoRoundMoney": 1,
      "EnableBluetoothPrint": 0,
      "EnableWifiPrint": 0
    }
  }
}
```

### `PUT /settings/{key}`

**Role:** TenantAdmin với tenant setting. Giá trị phải validate theo schema của key. Các key cấu hình cửa hàng gồm:

- `StoreProfile`: `{ "storeName": "...", "address": null, "phone": null, "taxCode": null, "bankTransfer": { "bin": "970436", "accountNumber": "000123456789", "accountName": "TEST ACCOUNT" } }`. `bankTransfer` tùy chọn, bỏ cả property hoặc đặt null để xóa; nếu có object phải đủ ba field, BIN đúng 6 chữ số ASCII, tài khoản 6–34 chữ số ASCII (giữ số 0 đầu), tên không trống/tối đa 100 ký tự/không control characters. Không nhận object thiếu.
- `ReceiptTemplate`: `{ "templateId": "Standard", "showStoreAddress": true, "showTaxBreakdown": true, "footerText": null, "layout": { ... } }`; `layout` tùy chọn, theo `ReceiptLayout` trong OpenAPI. Không có layout vẫn dùng Standard legacy.
- `ProductDisplay`: `{ "layout": "Grid", "sortBy": "Name", "showSku": true, "showBarcode": false, "showCategory": true }`; `layout` là `Grid`/`List`, `sortBy` là `Name`/`Sku`.

Request dùng `{ "value": <value theo schema>, "expectedVersion": 2 }`, lấy `expectedVersion` từ `settingVersions[key]` của tenant hiện tại; `0` khi chưa có tenant override, kể cả khi effective value đến từ global setting. `settingsVersion` là aggregate, không phải concurrency token dùng cho PUT. Sau PUT dùng version trả về cho lần ghi tiếp theo; conflict trả `409 ERR_VERSION_CONFLICT`, không tự retry ghi đè.

`settingVersions` là metadata additive cho các key `StoreProfile`, `ReceiptTemplate`, `ProductDisplay`, `AutoRoundMoney`, `EnableBluetoothPrint`, `EnableWifiPrint`. Editor không được đoán version của năm key hiện có nếu thiếu metadata. Riêng Wi-Fi bổ sung sau: API/cache cũ thiếu key thì đọc disabled/version 0; backend cũ vẫn không hỗ trợ ghi key mới. Triển khai backend trước mobile editor.

Thay đổi áp dụng cho tenant hiện tại và được trả về bởi `/settings/effective`, không sửa invoice đã đồng bộ. TenantAdmin được ghi feature flags `AutoRoundMoney`, `EnableBluetoothPrint`, `EnableWifiPrint`; Wi-Fi mặc định false, là quyền in cho tenant; Bluetooth chỉ lưu tùy chọn, chưa có transport. SuperAdmin quản lý global feature flags qua `/platform/feature-flags`.

#### Receipt layout version 1 và 2

Version 2 giữ contract version 1 và thêm `header: { "left": "...", "right": "..." }`, mỗi chuỗi tối đa 1000 ký tự; header chỉ hợp lệ ở v2. Khi một cột không rỗng, hai cột thay nhóm thông tin cửa hàng mặc định. Editor không có header vẫn ghi v1. Template không có layout mặc định A4 ở mobile mới; layout cũ có giấy rõ ràng được giữ nguyên. In qua PC yêu cầu A4.

Bản in thật luôn có số hóa đơn + Code128, ngày tạo, dòng khách hàng để ghi tay và chữ ký, dù mẫu cũ tắt các khối này. QR nhận tiền tùy bankTransfer, chỉ VND nguyên dương tối đa 12 chữ số, không áp dụng Reverse; không tag nội dung chuyển khoản/tên người gửi, không gọi API ngân hàng và không ghi nhận thanh toán. Xem [compatibility matrix](api_compatibility_matrix.md) trước rollout: editor cũ có thể xóa bankTransfer hoặc không đọc được v2.

`layout` gồm `version: 1`, `paper`, `showSku`, `showUnit`, `blocks`. `paper` gồm preset `Roll58`/`Roll80`/`A0`–`A10`/`Custom`, `widthMm`, `heightMm` (null chỉ với giấy cuộn), `marginMm`. A-series dùng kích thước ISO 216 dạng đứng; preset và kích thước phải khớp. Giới hạn rộng 25–1500 mm, cao 25–2000 mm, lề 0–10 mm; vùng in sau hai lề tối thiểu 10 mm mỗi chiều.

`blocks` có 2–40 phần tử, ID không rỗng/tối đa 80 ký tự và duy nhất, thứ tự array là thứ tự in. Built-in type chỉ xuất hiện tối đa một lần; `Items`, `Total` bắt buộc xuất hiện và enabled=true. Built-in nhận `{ id, type, enabled }`. `Text` nhận thêm `text` (1–1000 ký tự, không chỉ khoảng trắng) và `align` (`Left`, `Center`, `Right`). `Service` nhận `label` (1–120 ký tự, không chỉ khoảng trắng), `hideWhenZero`; tối đa 10 service. Đây là plain text, không HTML/script, không nội suy code.

Service block chỉ định nghĩa danh mục, **không nhận `amount`, giá mặc định hoặc số tiền mẫu**. Quy tắc cho bước invoice sau: tiền dịch vụ nhập riêng từng hóa đơn, mặc định 0, không âm, cộng tổng; ẩn khi 0 mặc định bật. Chưa thay đổi schema/formula của invoice v1, chưa hỗ trợ sync khoản phí thật. Việc ẩn khối chỉ ảnh hưởng in, không thay đổi tổng tiền hoặc lịch sử invoice.

Các field legacy được giữ để tương thích đọc; khi có layout, renderer mới dùng layout làm nguồn bố cục, `footerText` vẫn là nội dung khối Footer. PUT thay thế toàn bộ value của ReceiptTemplate: client phải giữ layout và mọi field đang dùng, không gửi patch một phần. Xem deployment gate trong compatibility matrix để tránh editor cũ ghi mất layout.

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
  "basePriceText": "25000",
  "listedPrice": 25000.0000,
  "listedPriceText": "25000",
  "calculationQuantity": 1.0000,
  "calculationQuantityText": "1",
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
| POST | `/products` | `CreateProductRequest` + `Idempotency-Key` | `AcceptedCommandResponse` | 202 |
| GET | `/products/{productId}/variants` | query `includeInactive` | `ProductVariant[]` | 200 |
| GET | `/products/{productId}/variants/{variantId}` | path `productId,variantId` | `ProductVariant` | 200 |
| POST | `/products/{productId}/variants` | TenantAdmin; `CreateProductVariantRequest` | `ProductVariant` | 201 |
| PATCH | `/products/{productId}/variants/{variantId}` | TenantAdmin; `UpdateProductVariantRequest` + `expectedVersion` | `ProductVariant` | 200 |
| POST | `/products/{productId}/variants/{variantId}/deactivate` | TenantAdmin; `expectedVersion` | Inactive `ProductVariant` | 200 |
| POST | `/products/{productId}/variants/{variantId}/activate` | TenantAdmin; `expectedVersion` | Active `ProductVariant`; parent/siblings unchanged | 200 |
| PATCH | `/products/{id}` | `UpdateProductRequest` + `expectedVersion` | `Product` | 200 |
| POST | `/products/{id}/deactivate` | `expectedVersion` | `Product` | 200 |
| POST | `/products/{id}/activate` | TenantAdmin; `expectedVersion` | `Product`; all variants active | 200 |
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
| PATCH | `/categories/{id}/status` | `isActive,expectedVersion` | `Category` | 200 |
| DELETE | `/categories/{id}` | query `expectedVersion` | Không có body; chỉ category không có product/con | 204 |
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
