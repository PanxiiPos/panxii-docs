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
| `ERR_INVALID_REQUEST` | 400 | No | `errors.invalidRequest` | JSON/schema sai |
| `ERR_INVALID_PAGING` | 400 | No | `errors.invalidPaging` | Page hoặc page size không hợp lệ |
| `ERR_INVALID_ENUM` | 400 | No | `errors.invalidEnum` | Enum không hợp lệ |
| `ERR_INVALID_DECIMAL` | 400 | No | `errors.invalidDecimal` | Precision/range sai |
| `ERR_UNAUTHENTICATED` | 401 | No | `errors.unauthenticated` | Request chưa xác thực |
| `ERR_INVALID_CREDENTIALS` | 401 | No | `errors.invalidCredentials` | Tenant/user/password không hợp lệ |
| `ERR_ACCESS_DENIED` | 403 | No | `errors.accessDenied` | Không đủ quyền |
| `ERR_NOT_FOUND` | 404 | No | `errors.notFound` | Route/resource không tồn tại |
| `ERR_CONFLICT` | 409 | No | `errors.conflict` | Request xung đột trạng thái hiện tại |
| `ERR_TENANT_INACTIVE` | 422 | No | `errors.tenantInactive` | Tenant bị khóa |
| `ERR_DEVICE_REVOKED` | 401 | No | `errors.deviceRevoked` | Device bị revoke |
| `ERR_PRODUCT_NOT_FOUND` | 404 | No | `errors.productNotFound` | Không thấy product |
| `ERR_PRODUCT_INACTIVE` | 422 | No | `errors.productInactive` | Product ngừng bán |
| `ERR_PRODUCT_DELETE_REQUIRES_DEACTIVATION` | 409 | No | `errors.productDeleteRequiresDeactivation` | Product có lịch sử, chỉ được deactivate |
| `ERR_DUPLICATE_SKU` | 409 | No | `errors.duplicateSku` | SKU trùng |
| `ERR_DUPLICATE_BARCODE` | 409 | No | `errors.duplicateBarcode` | Barcode trùng |
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
| `ERR_ADMIN_PIN_INVALID` | 401 | No | `errors.adminPinInvalid` | PIN sai |
| `ERR_IDEMPOTENCY_KEY_MISMATCH` | 400 | No | `errors.idempotencyMismatch` | Header/body lệch |
| `ERR_IDEMPOTENCY_KEY_REUSED` | 409 | No | `errors.idempotencyKeyReused` | Key dùng payload khác |
| `ERR_SYNC_PAYLOAD_MISMATCH` | 409 | No | `errors.syncPayloadMismatch` | Payload retry khác |
| `ERR_SYNC_CONFLICT` | 409 | No | `errors.syncConflict` | Xung đột version |
| `ERR_CURSOR_EXPIRED` | 409 | No | `errors.cursorExpired` | Cursor hết hạn |
| `ERR_TENANT_NOT_FOUND` | 404 | No | `errors.tenantNotFound` | Không thấy tenant |
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
