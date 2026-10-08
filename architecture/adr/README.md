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
