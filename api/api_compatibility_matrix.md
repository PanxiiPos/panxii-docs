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
- Breaking change phải dùng `/api/v2` hoặc expand/contract.
- Mobile unknown enum phải map `Unknown`.
- Server giữ field cũ trong deprecation window.

## 3. CI checks

- Lint OpenAPI.
- Diff current spec với baseline release.
- Consumer contract test bằng model mobile cũ và mới.
- Kiểm tra error catalog có đủ `vi.json`/`en.json` keys.
- Kiểm tra migration chạy trong lúc app cũ vẫn sync.

## 4. Release evidence

Mỗi release lưu OpenAPI hash, compatibility result, minimum native version, deprecation list và rollback decision.
