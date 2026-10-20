# API COMPATIBILITY MATRIX

## 1. Supported versions

| Mobile | API | Status |
|---|---|---|
| 2.x | v1 | Supported |
| 1.x | v1 | Deprecation window cần chốt |
| 3.x | v1/v2 | Chỉ sau compatibility test |

## 2. Contract rules

- Additive optional response field: compatible.
- `GET /settings/effective` bổ sung `settingVersions` (version tenant override theo từng key, `0` nếu chưa override). Deploy API này trước mobile có editor cấu hình; client cũ tiếp tục dùng `settingsVersion` cho cache. Editor mới không được suy diễn version ghi từ `settingsVersion` tổng hoặc tự retry với version mới khi `409`.
- `ReceiptTemplate.layout` là object tùy chọn version 1 cho giấy và khối in có thứ tự; template legacy không có layout vẫn hợp lệ. Deploy backend validator mới trước mobile designer; backend cũ trả 400 khi nhận layout. Chỉ bật designer khi các client được phép sửa mẫu đã hỗ trợ giữ layout: PUT là full replacement nên editor legacy có thể xóa layout khi lưu. Renderer legacy chỉ dùng field cũ, không bảo đảm thể hiện khối tùy chỉnh; cần phối hợp rollout reader/writer, không tự tuyên bố bản in cũ giống preview mới.
- Service blocks mới chỉ là định nghĩa trong mẫu in, không thêm tiền phí vào invoice payloadVersion=1. Mở rộng phí giao dịch cần contract/payload version và compatibility tests cho invoice totals, offline sync, reverse/correction trước rollout.
- Required request field, field type, enum meaning, envelope hoặc error code đổi: breaking.
- Thay đổi role được phép thực hiện command là behavioral breaking change; không xem việc trả `403` thay cho hành vi trước đây là additive.
- Chuyển unknown request fields từ ignore sang reject là behavioral breaking change với client gửi extra fields; kiểm tra request thực tế trước rollout.
- Breaking change phải dùng `/api/v2` hoặc expand/contract.
- Changing create/sync responses from `201`/`200` to `202 Queued` is behavioral breaking: release the mobile SignalR/operation-state consumer before enabling the queued endpoints for production traffic.
- Mobile unknown enum phải map `Unknown`.
- Server giữ field cũ trong deprecation window.
- Invoice status contract target tách business `InvoiceStatus` (`Active`, `Reversed`) khỏi client-local `SyncStatus`. Thay đổi semantics của response field `status` từ giá trị hiện hành là breaking change; nếu v1 đã phát hành, cần expand/contract hoặc API v2 và migration có phối hợp, không rollout âm thầm.

### Paging migration trong API v1

- Mobile mới chỉ dùng cursor keyset; request trang đầu phải gửi `cursor=` rỗng, các trang sau dùng nguyên `nextCursor`.
- Giữ `page` offset ở chế độ deprecated cho client v1 cũ. Nếu query có cả `page` và `cursor`, trả `400 ERR_INVALID_PAGING`.
- Cursor response dùng `nextCursor`, `hasMore`, `pageSize`; legacy page response tiếp tục trả `page`, `hasMore`, `pageSize`.
- Không gỡ `page` khỏi API v1 cho đến khi hết deprecation window, đã kiểm tra telemetry/version adoption và có compatibility approval. Nếu không thể giữ expand/contract, chuyển đổi phải phát hành qua API v2.

### RBAC contract change trong v1

Business rule v1 cho phép Staff tìm/đọc/tạo customer nhưng không sửa/xóa; TenantAdmin quản lý customer theo RBAC. Chỉ TenantAdmin được reverse/correction invoice. Contract cũ cho Staff reverse bằng PIN; nếu client/release đã dùng quyền này, thu hồi quyền là behavioral breaking change và phải đi theo versioning/deprecation policy, không đổi âm thầm. Quyền Staff tạo customer được giữ nguyên.

## 3. CI checks

- Lint OpenAPI.
- Diff current spec với baseline release.
- Consumer contract test bằng model mobile cũ và mới.
- Kiểm tra error catalog có đủ `vi.json`/`en.json` keys.
- Kiểm tra migration chạy trong lúc app cũ vẫn sync.

## 4. Release evidence

Mỗi release lưu OpenAPI hash, compatibility result, minimum native version, deprecation list và rollback decision.
