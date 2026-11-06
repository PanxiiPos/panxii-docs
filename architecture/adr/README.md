# ARCHITECTURE DECISION RECORDS

Thư mục này lưu các quyết định kiến trúc có ảnh hưởng lâu dài. Mỗi ADR gồm: context, decision, alternatives, consequences, status và date.

## Quy tắc

- Không sửa lịch sử ADR đã accepted; tạo ADR mới để thay thế.
- Một ADR chỉ giải quyết một quyết định.
- Code, migration và tài liệu liên quan phải link tới ADR.
- ADR chưa chốt không được coi là contract production.

## Danh sách ADR cần chốt

| ID | Quyết định | Trạng thái |
|---|---|---|
| ADR-001 | UUID `CHAR(36)` hay `BINARY(16)` | Proposed |
| ADR-002 | Cấp InvoiceNumber client/server | Proposed |
| ADR-003 | Cho phép reverse offline hay không | Proposed |
| ADR-004 | Conflict strategy cho master data offline | Proposed |
| ADR-005 | Payment transactions và phương thức thanh toán | Proposed |
| ADR-006 | Product price history | Proposed |
| ADR-007 | Xử lý outbox pending khi device bị revoke | Proposed |
| ADR-008 | Effective feature flag precedence | Accepted/verify |
| ADR-009 | Role được phép tạo customer và reverse invoice | Accepted |
| ADR-010 | Queued POS commands và realtime entity snapshots | Accepted |
| ADR-011 | Product gốc và các biến thể bán được | Accepted |
| ADR-012 | In A4 qua PC Windows và QR chỉ dẫn chuyển khoản | Accepted |
| ADR-013 | Tự tìm lại PC in khi DHCP đổi IPv4 | Accepted |
| ADR-014 | Mã hàng và số hóa đơn ngắn, giữ UUID identity | Accepted |
| ADR-015 | [Category-scoped sale-price rounding](ADR-015-category-sale-price-rounding.md) | Accepted |

## Mẫu ADR

```markdown
# ADR-NNN: Tên quyết định

- Status: Proposed | Accepted | Superseded
- Date: YYYY-MM-DD

## Context
## Decision
## Alternatives considered
## Consequences
## Related documents
```
