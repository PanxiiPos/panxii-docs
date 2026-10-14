# API COMPATIBILITY MATRIX

## 1. Supported versions

| Mobile | API | Status |
|---|---|---|
| 2.x | v1 | Supported |
| 1.x | v1 | Deprecation window cần chốt |
| 3.x | v1/v2 | Chỉ sau compatibility test |

## 2. Contract rules

- Additive optional response field: compatible.
- Required request field, field type, enum meaning, envelope hoặc error code đổi: breaking.
- Thay đổi role được phép thực hiện command là behavioral breaking change; không xem việc trả `403` thay cho hành vi trước đây là additive.
- Chuyển unknown request fields từ ignore sang reject là behavioral breaking change với client gửi extra fields; kiểm tra request thực tế trước rollout.
- Breaking change phải dùng `/api/v2` hoặc expand/contract.
- Changing create/sync responses from `201`/`200` to `202 Queued` is behavioral breaking: release the mobile SignalR/operation-state consumer before enabling the queued endpoints for production traffic.
- Mobile unknown enum phải map `Unknown`.
- Server giữ field cũ trong deprecation window.

### RBAC contract change trong v1

Business rule chuẩn hóa Staff thành read-only đối với customer và không được reverse/correction invoice; TenantAdmin là role duy nhất được ghi customer hoặc thực hiện reverse/correction. Contract cũ từng cho Staff tạo customer và reverse bằng PIN. Trước release, cập nhật baseline v1 và kiểm tra mobile không còn phụ thuộc các thao tác này. Nếu đã có client/release dùng quyền cũ, xử lý như breaking change theo versioning/deprecation policy thay vì âm thầm đổi behavior.

## 3. CI checks

- Lint OpenAPI.
- Diff current spec với baseline release.
- Consumer contract test bằng model mobile cũ và mới.
- Kiểm tra error catalog có đủ `vi.json`/`en.json` keys.
- Kiểm tra migration chạy trong lúc app cũ vẫn sync.

## 4. Release evidence

Mỗi release lưu OpenAPI hash, compatibility result, minimum native version, deprecation list và rollback decision.
