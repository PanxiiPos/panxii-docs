# RELEASE MANAGEMENT

Hướng triển khai nội bộ tiết kiệm chi phí và setup OTA để thực hiện sau:
[Oracle Always Free, APK nội bộ và Shorebird Free](../infrastructure/internal_free_deployment.md).
Shorebird Android đã có release thử nội bộ `0.1.0+1` và đã cài trên thiết bị
thử nghiệm; chưa kiểm thử patch thật. Pipeline/cloud production chưa được
triển khai. Xem trạng thái chi tiết trong hướng dẫn, không dùng bản nội bộ
ký debug key để phân phối production.

## 1. Versioning

- API major breaking change dùng `/api/v2`.
- Backend/mobile dùng semantic versioning.
- `minimumNativeVersion` chặn app native quá cũ.
- OTA Shorebird chỉ dùng cho thay đổi tương thích native code.

## 2. Release tracks

| Track | Mục đích |
|---|---|
| Development | Local Docker, seed giả |
| Staging | E2E, load, migration, restore drill |
| Pilot | Một nhóm tenant/thiết bị giới hạn |
| Production | Rollout có monitoring và rollback plan |

## 3. Release checklist

- Business rule và ADR đã chốt.
- OpenAPI, error catalog và i18n keys đồng bộ.
- Database migration đã review/backup.
- Unit, integration, contract, offline E2E pass.
- Security scan và dependency scan pass.
- Docker image pin theo git SHA.
- Health check và smoke test pass.
- Release notes có breaking/behavior changes.
- Rollback code và data plan được xác nhận.

## 4. Mobile compatibility

Server phải hỗ trợ mobile cũ trong khoảng thời gian đã chốt. Thêm field là backward-compatible; đổi/xóa field hoặc enum cần version/compatibility handling.

## 5. Rollout

1. Deploy staging.
2. Chạy migration expand.
3. Chạy smoke/contract/load test.
4. Pilot rollout.
5. Theo dõi 5xx, sync failure, duplicate, DLQ, latency và crash rate.
6. Mở rộng production.
7. Chỉ contract breaking mới tạo major version.

## 6. Rollback

- Code rollback bằng image tag trước.
- Data rollback không mặc định là migration down; dùng forward fix hoặc restore theo DR.
- Mobile rollback phải giữ API compatibility hoặc force update về bản hỗ trợ.
- OTA patch phải có kế hoạch disable/release patch mới nếu lỗi.

## 7. Release notes bắt buộc

Ghi rõ API changes, migration, feature flags, minimum app version, known issues, monitoring metrics và hướng xử lý khi sync lỗi.
