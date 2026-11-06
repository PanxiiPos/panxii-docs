# API COMPATIBILITY MATRIX

## 1. Supported versions

| Mobile | API | Status |
|---|---|---|
| 2.x | v1 | Supported |
| 1.x | v1 | Deprecation window cần chốt |
| 3.x | v1/v2 | Chỉ sau compatibility test |

## 2. Contract rules

- In Wi-Fi: deploy backend có `EnableWifiPrint`/`settingVersions.EnableWifiPrint`, `StoreProfile.bankTransfer` và layout v2 trước mobile mới. API/cache cũ thiếu Wi-Fi được mobile đọc false/version 0, không ngầm bật quyền. Không cần migration invoice.
- Layout v2 thêm header hai cột; mobile mới đọc v1/v2, không header vẫn ghi v1. Client cũ có thể từ chối v2; PUT full replacement của editor cũ có thể xóa header/bankTransfer. Chỉ bật chỉnh sửa cấu hình này sau khi mọi editor được phép ghi đã nâng cấp hoặc đã chặn editor cũ bằng minimum native version. Không tuyên bố tương thích ghi với editor legacy.
- Mobile mới mặc định A4 khi template chưa có layout; không đổi giấy đã lưu rõ ràng. In Wi-Fi chỉ A4; Bluetooth chưa có transport. Preview bản in thật bắt buộc số hóa đơn/barcode/ngày/khách hàng/chữ ký, không dùng các toggle legacy để ẩn thông tin bắt buộc. QR chỉ là chỉ dẫn chuyển khoản, không đổi invoice/payment contract.

- Additive optional response field: compatible.
- Category pricing adds optional `roundSalePriceToThousands`, default false.
  An omitted update flag preserves the stored policy. Deploy the additive
  migration/API/Worker before enabling a category. Enabled categories accept
  either the legacy four-decimal product price or the new rounded price, but
  persist the server-calculated policy price; unrelated mismatches remain invalid.
  Flutter uses fixed-point arithmetic with the same half-up threshold and minimum
  1000 VND. Refresh category caches after enabling the policy; publishing the
  mobile patch does not deploy the backend or enable the flag.
- Enabling a category flag alone does not rewrite existing prices or invoice
  snapshots. Moving a family between pricing policies recalculates its current
  catalog prices atomically, including inactive variants. Existing invoice sync
  checks each submitted unit price against the current catalog: an offline sale
  created before repricing can be rejected with `ERR_INVALID_FIELD`, not silently
  charged at a different price. Before enabling/repricing or moving a family,
  require every selling device to sync its pending sales and resolve open drafts.
  Empty server queues do not prove offline devices have no pending sales.
  Do not lower this invoice validation or rewrite historical invoices as a
  rollout workaround. A code/migration-only deploy keeps all new flags false.
- Product/variant rollout: backend API and database migration must be deployed before a mobile release calls `/products/{productId}/variants`. Existing v1 clients may continue using the product fields and omit `productVariantId` on invoice sync; the server resolves that legacy line to the default variant whose ID equals `productId`. New clients must persist and send the explicitly selected `productVariantId`; never infer a variant from SKU, display name, or price.
- Bulk-price accepts product/base targets, variant targets, or both; existing product-only requests remain valid. The response adds variant results while retaining product results, so clients must ignore unknown response fields. A mixed request applies all valid updates atomically, with product/default-variant aliases adjusted once.
- `productVariantId` and `variantNameSnapshot` are optional in the invoice request for legacy compatibility. Invoice responses add the resolved `productVariantId`, `variantNameSnapshot`, and `variantExtraDataSnapshot`; readers must tolerate their absence on records from older/imported data.
- A client that needs variant sales must not enable that workflow against an API version without the variant endpoints. Roll out database and compatible backend first, then the mobile reader/writer; keep legacy product fields until supported clients have moved.
- `GET /settings/effective` bổ sung `settingVersions` (version tenant override theo từng key, `0` nếu chưa override). Deploy API này trước mobile có editor cấu hình; client cũ tiếp tục dùng `settingsVersion` cho cache. Editor mới không được suy diễn version ghi từ `settingsVersion` tổng hoặc tự retry với version mới khi `409`.
- `ReceiptTemplate.layout` là object tùy chọn version 1 cho giấy và khối in có thứ tự; template legacy không có layout vẫn hợp lệ. Deploy backend validator mới trước mobile designer; backend cũ trả 400 khi nhận layout. Chỉ bật designer khi các client được phép sửa mẫu đã hỗ trợ giữ layout: PUT là full replacement nên editor legacy có thể xóa layout khi lưu. Renderer legacy chỉ dùng field cũ, không bảo đảm thể hiện khối tùy chỉnh; cần phối hợp rollout reader/writer, không tự tuyên bố bản in cũ giống preview mới.
- Phí dịch vụ dùng invoice payloadVersion=2, thêm `serviceCharges` (ID/nhãn/số tiền) và `serviceTotal`; v1 tiếp tục không phí. Deploy migration + API/worker mới trước mobile gửi v2. Response thêm snapshots là additive; client cũ có thể không render breakdown nên không bảo đảm bản in/chi tiết đầy đủ phí. Với correction legacy không được âm thầm làm mất phí invoice gốc.
- Receipt `GET /invoices/operations/{operationId}` là additive, tenant/actor scoped; Completed xác nhận invoice commit, Rejected mang errorCode. Mobile mới dùng receipt cho queued/uploading; API cũ 404 fallback GET invoice và retry cùng operation UUID/payload. Không đổi HTTP 202 thành success cuối hoặc tạo operation mới khi chỉ mất response.
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
