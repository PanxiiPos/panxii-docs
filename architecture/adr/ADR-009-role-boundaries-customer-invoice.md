# ADR-009: Role boundaries for customer writes and invoice reversal

- Status: Accepted
- Date: 2026-10-12

## Context

The approved business rules assign customer administration and invoice reversal/correction to TenantAdmin. Earlier API documentation allowed Staff to create customers and reverse invoices by supplying an Admin PIN, which widened the Staff role beyond the business rules.

## Decision

- Staff may search/read customers and create, sync, read, and print invoices within their tenant.
- Staff may not create, update, delete customers, or reverse/correct invoices. Supplying an Admin PIN does not grant these permissions.
- TenantAdmin is the only tenant role allowed to write customer data or execute invoice reverse/correction use cases.
- Authorization is based on the authenticated role and tenant context; actor identity is never accepted from a request body.

## Alternatives considered

- Keep Staff customer creation and invoice reversal behind an Admin PIN.
- Let Staff submit reversal requests while TenantAdmin approves them in a separate workflow.

These alternatives require business rules and approval flows not present in the approved contract, so they are not part of API v1.

## Consequences

- Customer write and invoice reverse/correction endpoints require TenantAdmin; Staff requests return `403 ERR_ACCESS_DENIED`.
- `ReverseInvoiceRequest` does not contain `adminUserId` or `adminPin`.
- Customer hard-delete is allowed only when no invoice/history references exist; otherwise TenantAdmin deactivates the customer.
- Treat this as a behavioral breaking change if an already released client depends on the former permissions; verify the release baseline and compatibility policy before rollout.

## Related documents

- [business_rule.md](../../business_rule.md)
- [api_contract.md](../../api/api_contract.md)
- [openapi.yaml](../../api/openapi.yaml)
- [api_compatibility_matrix.md](../../api/api_compatibility_matrix.md)
- [role_based_test_cases.md](../../testing/role_based_test_cases.md)
