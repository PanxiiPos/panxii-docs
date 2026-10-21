# MIGRATION VÀ SEED STRATEGY

## 1. Nguyên tắc

- Schema thay đổi qua EF Core migration, không sửa database production thủ công.
- Migration phải backward-compatible với phiên bản app đang được hỗ trợ.
- Migration production chạy bằng pipeline và migration user riêng.
- Không seed password/PIN/secret cố định trong source code.

## 2. Phân loại seed

| Loại | Môi trường | Quy tắc |
|---|---|---|
| Reference seed | All | Role, enum mapping, feature flag definitions; idempotent |
| Development seed | Local/test | Dữ liệu giả, không chạy production |
| Tenant bootstrap | Production | Tạo tenant và admin qua command có audit |
| Secret bootstrap | Production | Docker secret/secret manager, không nằm trong migration |

## 3. Quy trình migration

1. Tạo migration từ model.
2. Review SQL tạo ra, index, FK, lock và thời gian chạy.
3. Test trên database snapshot có dữ liệu lớn.
4. Backup trước production.
5. Chạy migration trong maintenance/expand phase.
6. Deploy code tương thích.
7. Chạy contract/health smoke test.
8. Xóa cột cũ ở migration riêng sau thời gian deprecation.

## 4. Expand/contract

Không đổi tên/xóa cột ngay khi mobile cũ còn hoạt động. Thêm cột nullable hoặc bảng mới, deploy code đọc/ghi tương thích, backfill, theo dõi, rồi mới remove.

### 4.1. Chuyển catalog product phẳng sang product/variant

Khi triển khai mô hình `Product`/`ProductVariant` theo [database_design.md](../database/database_design.md), thực hiện theo các pha tương thích:

1. **Expand:** migration `AddProductVariants` tạo `product_variants`; thêm `ProductVariantId` nullable vào `invoice_details`; thêm alternate key/FK cùng tenant. Migration `AddInvoiceVariantSnapshot` thêm `VariantNameSnapshot` và JSON `VariantExtraDataSnapshot`. Chưa xóa hoặc đổi nghĩa các cột hiện có trên `products`.
2. **Backfill:** với mỗi product cũ, tạo một variant giữ nguyên SKU, barcode, unit, base/cost price, tax rate, active state, version và audit fields. Có thể dùng `Product.Id` làm `ProductVariant.Id` trong backfill để giữ ánh xạ identity ổn định; `ProductId` vẫn trỏ product gốc. Gán `invoice_details.ProductVariantId = ProductId` cho dữ liệu cũ có product reference.
3. **Deploy tương thích:** backend/API mới quản lý variant và đọc/ghi variant ID; client mới gửi `ProductVariantId`, còn client cũ được map về default variant có ID bằng `ProductId`. Mobile/offline cache phải được cập nhật riêng trước khi dùng variant offline.
4. **Verify:** kiểm tra số lượng, SKU uniqueness, các barcode duplicate, giá/version/audit khớp và mọi invoice detail lịch sử có variant mapping. Migration tạo index không unique cho barcode để không làm mất dữ liệu legacy; API kiểm tra uniqueness khi ghi. Chỉ nâng index barcode lên unique sau khi báo cáo và xử lý duplicate có audit.
5. **Contract riêng:** sau khi hết thời gian hỗ trợ client cũ và đã xác nhận đồng bộ, bỏ các cột SKU/barcode/unit/price/cost/tax khỏi `products`. Không gộp bước này vào migration expand.

Migration hiện thực hiện backfill theo một lệnh SQL duy nhất và phụ thuộc EF migration history để chỉ chạy một lần. Với production có lượng dữ liệu lớn, cần rehearsal/backup trước khi apply; nếu vượt thời gian bảo trì, thay bằng job backfill riêng có checkpoint và idempotency. Không tự gộp hoặc xóa product trùng; cần xử lý conflict bằng quy trình dữ liệu có audit.

## 5. Rollback

Migration rollback chỉ dùng khi đã test. Với migration dữ liệu không đảo ngược được, rollback bằng forward migration hoặc restore backup theo runbook. Không hứa rollback code là rollback data.

## 6. CI gate

- `dotnet ef migrations script --idempotent` phải sinh được.
- Không có destructive SQL ngoài migration được phê duyệt.
- Test tenant filter, unique key, invoice immutability.
- Startup không tự chạy migration production nếu chưa có policy rõ ràng.
