# ADR-011: Product gốc và các biến thể bán được

- Status: Accepted
- Date: 2026-10-21

## Context

Một sản phẩm có thể có nhiều lựa chọn bán được với SKU, barcode, đơn vị hoặc giá khác nhau. Ví dụ sắt hộp 30x60 của Hòa Phát và Việt Nhật có cùng độ dày nhưng giá khác nhau. Đặt nhiều mức giá trong `ExtraDataJson` khiến việc tìm SKU, cập nhật giá, đồng bộ và snapshot hóa đơn không có ràng buộc dữ liệu rõ ràng.

## Decision

- `Product` là sản phẩm gốc/nhóm hàng, giữ tên chung, danh mục và thuộc tính chung.
- `ProductVariant` là mặt hàng bán được; SKU, barcode, unit, BasePrice, CostPrice, TaxRate, trạng thái bán và thuộc tính riêng thuộc về variant.
- Mỗi product gốc phải có ít nhất một variant active; kiểm tra invariant trong application transaction.
- `InvoiceDetail` tham chiếu variant đã bán và giữ snapshot SKU, tên sản phẩm/variant, unit, thuộc tính cần hiển thị và giá tại thời điểm bán.
- `ExtraDataJson` dùng cho thuộc tính bổ sung; không dùng thay SKU/giá hay dữ liệu cần FK, lọc/báo cáo thường xuyên hoặc tính toán tài chính.
- Thay đổi schema triển khai theo expand/backfill/compatible-deploy/verify/contract; giữ các cột phẳng cũ cho đến khi client cũ không còn được hỗ trợ.

## Alternatives considered

- Giữ mỗi SKU là một Product riêng và chỉ thêm nhóm UI tùy chọn: tránh thay đổi sâu schema nhưng không mô hình hóa quan hệ gốc-biến thể, thuộc tính chung và snapshot một cách thống nhất.
- Lưu giá và lựa chọn trong `ExtraDataJson`: không có unique/index/FK rõ ràng và dễ tạo dữ liệu không nhất quán.

## Consequences

- Catalog, bulk-price, search, offline cache, sync payload và invoice line phải nhận diện `ProductVariantId`; bulk-price có thể nhận độc lập product/base targets và variant targets trong cùng transaction.
- SKU có unique index theo tenant trên variants; API kiểm tra SKU/barcode không trùng giữa cả product legacy và variants. Barcode index hiện không unique để giữ được barcode trùng trong dữ liệu legacy; cần dọn conflict trước khi enforce ở database.
- Variant mặc định được backfill/tạo mới với `ProductVariant.Id = Product.Id`. Client cũ không gửi variant ID sẽ được map tường minh về default variant; client mới gửi `ProductVariantId`.
- Migration expand và các endpoint variant đã được triển khai ở backend. Bulk-price nhận `productIds` và/hoặc `variantIds`, kiểm tra version riêng theo từng loại và cập nhật atomic; chọn product cùng default variant chỉ điều chỉnh giá dùng chung một lần. Mobile/offline cache, variant-aware search và contract bỏ cột legacy vẫn là phần tiếp theo.
- Dữ liệu invoice đã đồng bộ tiếp tục bất biến; migration chỉ backfill tham chiếu/snapshot còn thiếu theo quy trình đã audit.

## Related documents

- [database_design.md](../../database/database_design.md)
- [migration_seed_strategy.md](../../backend/migration_seed_strategy.md)
- [business_rule.md](../../business_rule.md)
