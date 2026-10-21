# ERROR CODE CATALOG

Error code là public contract giữa Backend và Flutter. Backend chỉ trả code/args; mobile dịch bằng `easy_localization`.

## 1. Quy ước

- Prefix `ERR_`, uppercase snake case.
- Code không đổi sau khi phát hành.
- `args` chỉ chứa dữ liệu cần render, không chứa secret/PII không cần thiết.
- Mỗi code có HTTP status, retry policy và i18n key.
- Xóa code phải qua API versioning/ADR.

## 2. Catalog v1

| Code | HTTP | Retry | i18n key | Ý nghĩa |
|---|---:|---:|---|---|
| `ERR_INVALID_REQUEST` | 400 | No | `errors.invalidRequest` | JSON không parse được hoặc body không đúng request object |
| `ERR_REQUIRED_FIELD` | 400 | No | `errors.requiredField` | Thiếu/null/rỗng field bắt buộc |
| `ERR_UNKNOWN_FIELD` | 400 | No | `errors.unknownField` | Request có JSON property không khai báo trong schema |
| `ERR_INVALID_FIELD` | 400 | No | `errors.invalidField` | Type, format hoặc constraint của field sai |
| `ERR_INVALID_UUID` | 400 | No | `errors.invalidUuid` | UUID sai định dạng |
| `ERR_INVALID_EMAIL` | 400 | No | `errors.invalidEmail` | Email sai định dạng hoặc vượt giới hạn |
| `ERR_INVALID_TIME_ZONE` | 400 | No | `errors.invalidTimeZone` | Time zone không hợp lệ |
| `ERR_INVALID_PAGING` | 400 | No | `errors.invalidPaging` | Page hoặc page size không hợp lệ |
| `ERR_INVALID_ENUM` | 400 | No | `errors.invalidEnum` | Enum không hợp lệ |
| `ERR_INVALID_DECIMAL` | 400 | No | `errors.invalidDecimal` | Precision/range sai |
| `ERR_PRICE_MUST_BE_POSITIVE` | 422 | No | `errors.priceMustBePositive` | Giá phải lớn hơn 0 |
| `ERR_PRICE_ADJUSTMENT_MUST_BE_POSITIVE` | 422 | No | `errors.priceAdjustmentMustBePositive` | Độ lớn tăng/giảm giá phải lớn hơn 0 |
| `ERR_INVALID_QUANTITY` | 422 | No | `errors.invalidQuantity` | Quantity của Sale phải lớn hơn 0 |
| `ERR_UNAUTHENTICATED` | 401 | No | `errors.unauthenticated` | Request chưa xác thực |
| `ERR_INVALID_CREDENTIALS` | 401 | No | `errors.invalidCredentials` | Tenant/user/password không hợp lệ |
| `ERR_ACCESS_DENIED` | 403 | No | `errors.accessDenied` | Không đủ quyền |
| `ERR_NOT_FOUND` | 404 | No | `errors.notFound` | Route/resource không tồn tại |
| `ERR_CONFLICT` | 409 | No | `errors.conflict` | Request xung đột trạng thái hiện tại |
| `ERR_TENANT_INACTIVE` | 422 | No | `errors.tenantInactive` | Tenant bị khóa |
| `ERR_DEVICE_REVOKED` | 401 | No | `errors.deviceRevoked` | Device bị revoke |
| `ERR_PRODUCT_NOT_FOUND` | 404 | No | `errors.productNotFound` | Không thấy product |
| `ERR_PRODUCT_VARIANT_NOT_FOUND` | 404 | No | `errors.productVariantNotFound` | Không thấy variant thuộc product trong tenant |
| `ERR_CATEGORY_NOT_FOUND` | 404 | No | `errors.categoryNotFound` | Không thấy category trong tenant |
| `ERR_DUPLICATE_CATEGORY_CODE` | 409 | No | `errors.duplicateCategoryCode` | Category code trùng trong tenant |
| `ERR_CATEGORY_PARENT_CYCLE` | 400 | No | `errors.categoryParentCycle` | Parent category tạo chu trình |
| `ERR_CATEGORY_NOT_FOUND` | 404 | No | `errors.categoryNotFound` | Không thấy category trong tenant |
| `ERR_DUPLICATE_CATEGORY_CODE` | 409 | No | `errors.duplicateCategoryCode` | Category code trùng trong tenant |
| `ERR_CATEGORY_PARENT_CYCLE` | 400 | No | `errors.categoryParentCycle` | Parent category tạo chu trình |
| `ERR_PRODUCT_INACTIVE` | 422 | No | `errors.productInactive` | Product ngừng bán |
| `ERR_PRODUCT_REQUIRES_ACTIVE_VARIANT` | 409 | No | `errors.productRequiresActiveVariant` | Product active phải còn ít nhất một variant active |
| `ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION` | 409 | No | `errors.productDeleteRequiresDeactivation` | Product có lịch sử, chỉ được deactivate |
| `ERR_DUPLICATE_SKU` | 409 | No | `errors.duplicateSku` | SKU trùng |
| `ERR_DUPLICATE_BARCODE` | 409 | No | `errors.duplicateBarcode` | Barcode trùng |
| `ERR_CUSTOMER_NOT_FOUND` | 404 | No | `errors.customerNotFound` | Không thấy customer trong tenant |
| `ERR_CUSTOMER_INACTIVE` | 422 | No | `errors.customerInactive` | Customer đã deactivate, không dùng cho invoice mới |
| `ERR_CUSTOMER_DELETE_REQUIRES_DEACTIVATION` | 409 | No | `errors.customerDeleteRequiresDeactivation` | Customer có invoice/history, chỉ được deactivate |
| `ERR_DUPLICATE_CUSTOMER_CODE` | 409 | No | `errors.duplicateCustomerCode` | Customer code trùng trong tenant |
| `ERR_STAFF_NOT_FOUND` | 404 | No | `errors.staffNotFound` | Không thấy Staff trong tenant |
| `ERR_STAFF_DELETE_REQUIRES_DEACTIVATION` | 409 | No | `errors.staffDeleteRequiresDeactivation` | Staff có lịch sử, chỉ được deactivate |
| `ERR_DUPLICATE_USERNAME` | 409 | No | `errors.duplicateUsername` | Username đã tồn tại trong tenant |
| `ERR_VERSION_CONFLICT` | 409 | No | `errors.versionConflict` | expectedVersion đã cũ |
| `ERR_SETTING_NOT_FOUND` | 404 | No | `errors.settingNotFound` | Setting không có trong registry |
| `ERR_CUSTOMER_PHONE_REQUIRED` | 422 | No | `errors.customerPhoneRequired` | Flag yêu cầu phone |
| `ERR_INVOICE_NOT_FOUND` | 404 | No | `errors.invoiceNotFound` | Không thấy invoice |
| `ERR_INVOICE_IMMUTABLE` | 422 | No | `errors.invoiceImmutable` | Invoice đã sync |
| `ERR_INVOICE_TOTAL_INVALID` | 422 | No | `errors.invoiceTotalInvalid` | Tổng không cân bằng |
| `ERR_INVOICE_ALREADY_REVERSED` | 409 | No | `errors.invoiceAlreadyReversed` | Đã reverse |
| `ERR_ADMIN_PIN_INVALID` | 401 | No | `errors.adminPinInvalid` | PIN sai; legacy, không dùng cho reverse invoice v1 |
| `ERR_IDEMPOTENCY_KEY_MISMATCH` | 400 | No | `errors.idempotencyMismatch` | Header/body lệch |
| `ERR_IDEMPOTENCY_KEY_REUSED` | 409 | No | `errors.idempotencyKeyReused` | Key dùng payload khác |
| `ERR_SYNC_PAYLOAD_MISMATCH` | 409 | No | `errors.syncPayloadMismatch` | Payload retry khác |
| `ERR_UNSUPPORTED_PAYLOAD_VERSION` | 400 | No | `errors.unsupportedPayloadVersion` | Payload version chưa được hỗ trợ |
| `ERR_SYNC_CONFLICT` | 409 | No | `errors.syncConflict` | Xung đột version |
| `ERR_CURSOR_EXPIRED` | 409 | No | `errors.cursorExpired` | Cursor hết hạn |
| `ERR_TENANT_NOT_FOUND` | 404 | No | `errors.tenantNotFound` | Không thấy tenant |
| `ERR_DUPLICATE_TENANT_CODE` | 409 | No | `errors.duplicateTenantCode` | Tenant code đã tồn tại |
| `ERR_TENANT_ADMIN_NOT_FOUND` | 404 | No | `errors.tenantAdminNotFound` | Không thấy TenantAdmin trong tenant |
| `ERR_TENANT_ADMIN_DELETE_REQUIRES_DEACTIVATION` | 409 | No | `errors.tenantAdminDeleteRequiresDeactivation` | TenantAdmin có lịch sử, chỉ được deactivate |
| `ERR_FEATURE_FLAG_NOT_FOUND` | 404 | No | `errors.featureFlagNotFound` | Feature flag không có trong registry |
| `ERR_FORCE_UPDATE_REQUIRED` | 426 | No | `errors.forceUpdateRequired` | App native quá cũ |
| `ERR_RATE_LIMITED` | 429 | Yes | `errors.rateLimited` | Quá tốc độ request |
| `ERR_SERVICE_UNAVAILABLE` | 503 | Yes | `errors.serviceUnavailable` | Dependency tạm unavailable |
| `ERR_PAYMENT_NOT_FOUND` | 404 | No | `errors.paymentNotFound` | Không thấy payment |
| `ERR_PAYMENT_AMOUNT_INVALID` | 422 | No | `errors.paymentAmountInvalid` | Số tiền payment không hợp lệ |
| `ERR_PAYMENT_ALREADY_CAPTURED` | 409 | No | `errors.paymentAlreadyCaptured` | Payment đã capture |
| `ERR_PAYMENT_PROVIDER_TIMEOUT` | 503 | Yes | `errors.paymentProviderTimeout` | Provider timeout |
| `ERR_PAYMENT_SIGNATURE_INVALID` | 401 | No | `errors.paymentSignatureInvalid` | Chữ ký callback sai |
| `ERR_PAYMENT_RECONCILIATION_FAILED` | 422 | No | `errors.paymentReconciliationFailed` | Đối soát thất bại |
| `ERR_PRINT_FAILED` | 503 | Yes | `errors.printFailed` | In thất bại, invoice không rollback |
| `ERR_LOCAL_DATABASE_RECOVERY_REQUIRED` | 503 | No | `errors.localDatabaseRecoveryRequired` | Cần khôi phục local database |
| `ERR_UNKNOWN` | 500 | No | `errors.unknown` | Lỗi ngoài dự kiến |

## 3. i18n minimum keys

`vi.json` và `en.json` phải có đủ các key ở cột `i18n key`. CI nên kiểm tra hai file có cùng key tree.

Mobile Login dùng mapping tập trung `AuthErrorCode.translationKeys` để giữ nguyên các alias của catalog, ví dụ `ERR_INVALID_CREDENTIALS -> errors.invalidCredentials`; không đổi wire code của API. Code client dùng namespace riêng `client.*` và key `errors.<client-code>`.

## 4. Lỗi local của mobile (không phải API wire code)

| Code | Xử lý | i18n key |
|---|---|---|
| `client.unknown` | Báo lỗi chung, cho thử lại; không hiển thị exception/raw server message | `errors.client.unknown` |
| `client.network_unavailable` | Báo mất kết nối/timeout; giữ form, cho đăng nhập lại thủ công | `errors.client.network_unavailable` |
| `client.auth.tenant_login_required` | Không lưu token hoặc vào tenant flow với tài khoản không được hỗ trợ | `errors.client.auth.tenant_login_required` |
| `client.display.read_failed` | Báo lỗi đọc preference; cho đọc lại/đặt lại | `errors.client.display.read_failed` |
| `client.display.write_failed` | Giữ cỡ chữ đã áp dụng trước đó; cho thử lưu lại | `errors.client.display.write_failed` |
| `client.display.value_invalid` | Báo giá trị lưu không hợp lệ; dùng cỡ chữ an toàn và cho đặt lại | `errors.client.display.value_invalid` |
