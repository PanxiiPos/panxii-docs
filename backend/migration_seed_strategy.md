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

## 5. Rollback

Migration rollback chỉ dùng khi đã test. Với migration dữ liệu không đảo ngược được, rollback bằng forward migration hoặc restore backup theo runbook. Không hứa rollback code là rollback data.

## 6. CI gate

- `dotnet ef migrations script --idempotent` phải sinh được.
- Không có destructive SQL ngoài migration được phê duyệt.
- Test tenant filter, unique key, invoice immutability.
- Startup không tự chạy migration production nếu chưa có policy rõ ràng.
