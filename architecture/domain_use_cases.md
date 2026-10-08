# DOMAIN MODEL VÀ USE CASE SPECIFICATION

## 1. Actors

| Actor | Phạm vi |
|---|---|
| SuperAdmin | Tenant lifecycle, global flags, platform support |
| TenantAdmin | Dữ liệu tenant, bulk price, reverse/correction, tenant flags |
| Staff | Tạo invoice, tìm product/customer, reverse khi có PIN Admin |
| Device Worker | Sync outbox, pull changes, xử lý revoke |

## 2. Core aggregates

- `Tenant`: status, timezone, currency, lifecycle.
- `User`: role, status, tenant membership.
- `UserDevice`: session, app version, revoke state.
- `Product`: price, active state, version.
- `Customer`: profile, tenant ownership.
- `Invoice`: immutable transaction content, status, reversal relationship.
- `AppSetting`: global/tenant effective configuration.

## 3. Invoice state machine

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Pending: Save local
    Pending --> Synced: Server applies
    Pending --> Rejected: Business rejection
    Synced --> Reversed: Reverse transaction
    Reversed --> [*]
    Rejected --> [*]
```

Invoice content is immutable after `Synced`. Only business status may change through the reverse use case.

## 4. Use case catalog

| ID | Use case | Actor | Transaction |
|---|---|---|---|
| UC-AUTH-01 | Login device | All users | Yes |
| UC-AUTH-02 | Refresh/revoke session | Authenticated | Yes |
| UC-TENANT-01 | Create/suspend tenant | SuperAdmin | Yes |
| UC-PROD-01 | Create/update product | TenantAdmin | Yes |
| UC-PROD-02 | Bulk update price | TenantAdmin | Yes |
| UC-INV-01 | Create invoice offline | Staff/Admin | Local transaction |
| UC-INV-02 | Sync invoice | Staff/Admin | MySQL + outbox |
| UC-INV-03 | Reverse invoice | Admin/PIN Staff | MySQL + outbox |
| UC-INV-04 | Correct invoice | TenantAdmin | Reverse + new invoice |
| UC-SYNC-01 | Pull changes | Device | Local transaction |
| UC-SET-01 | Resolve effective flags | Authenticated | Read |
| UC-SET-02 | Update feature flag | Admin | Yes |

## 5. Invariants

- Mọi tenant entity phải thuộc tenant context hiện tại.
- Sale quantity > 0; Reverse quantity < 0.
- `TotalAmount` phải cân bằng detail trong tolerance currency.
- Reverse phải có `OriginalInvoiceId` và chỉ được tạo một lần cho invoice gốc.
- Cùng `ClientOperationId` và payload khác nhau phải bị từ chối.
- Staff không được sửa giá gốc.
- `RequireCustomerPhone` phải được enforce ở server.
- Device revoked không được tiếp tục sync.

## 6. Acceptance criteria

Mỗi use case phải có request/response trong [API Contract](../api/api_contract.md), error code trong [Error Catalog](../api/error_codes.md), unit test và integration test theo [Testing Strategy](../testing/testing_strategy.md).
