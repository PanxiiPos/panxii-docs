# SECURITY, RBAC VÀ FEATURE FLAGS SAAS

## 1. Mô hình trust boundary

```text
Mobile/Web -> Nginx TLS -> ASP.NET API -> MySQL
                             |-> RabbitMQ
                             |-> SignalR
```

MySQL và RabbitMQ không mở public. Client không được truy cập trực tiếp database, broker hoặc SignalR group của tenant khác.

## 2. JWT và session

JWT tối thiểu có `sub`, `tenant_id`, `role`, `device_id`, `session_id`, `jti`, `iat`, `exp`. Không đặt thông tin nhạy cảm hoặc quyền thay đổi thường xuyên vào token quá dài.

- Access token ngắn hạn.
- Refresh token opaque, rotate mỗi lần refresh.
- Server chỉ lưu hash refresh token.
- Kiểm tra `user.Status`, `device.IsRevoked`, tenant status ở mỗi refresh và các command nhạy cảm.
- Logout/revoke phải invalidate session server-side.

## 3. RBAC matrix

| Resource/action | SuperAdmin | TenantAdmin | Staff |
|---|---:|---:|---:|
| Tạo/khóa tenant | Có | Không | Không |
| Global feature flags | Có | Không | Không |
| Tenant feature flags | Không | Có | Không |
| CRUD products/categories | Theo platform | Có | Đọc/tìm kiếm |
| Bulk update giá | Không trực tiếp | Có | Không |
| Tạo invoice | Không bắt buộc | Có | Có |
| Đọc invoice tenant | Platform support | Có | Có theo tenant |
| Reverse invoice | Không mặc định | Có | Chỉ PIN Admin |
| Sửa/xóa invoice synced | Không | Không | Không |
| Quản lý users tenant | Platform policy | Có | Không |
| Revoke device | Có platform | Có trong tenant | Không |

Authorization phải kiểm tra cả role và `TenantId`; role không đủ để cấp quyền cross-tenant.

## 4. Quy tắc tenant isolation

- Gán `TenantId` từ server context, không nhận để tin tưởng từ body.
- EF Core `Global Query Filter` cho mọi entity tenant-aware.
- Repository không cung cấp method query bỏ filter tùy tiện.
- Unique key nghiệp vụ luôn bao gồm `TenantId`.
- Kiểm tra ownership trước mọi update/delete/reverse.
- Audit lưu user, device, correlation id và lý do thay đổi.
- SuperAdmin path phải explicit, log và test riêng.

## 5. Password, PIN và secrets

- Password hash bằng Argon2id hoặc bcrypt với cost phù hợp.
- Admin PIN hash riêng, rate limit và lockout sau nhiều lần sai.
- Không log credential, refresh token, SQLCipher key hoặc PIN.
- SQLCipher key lưu secure storage của thiết bị; server không biết key local.
- Secrets production nằm trong Docker secrets hoặc secret manager, không commit repository.

## 6. Remote kill-switch

1. Admin chọn device và set `IsRevoked = true`.
2. API refresh/command kiểm tra trạng thái và trả `401 ERR_DEVICE_REVOKED`.
3. Mobile ngừng worker, đóng SQLite connection.
4. Xóa file SQLite và key local, xóa tokens, đưa user về login.
5. Ghi audit event, không cố gắng gửi outbox pending sau revoke.

Cần quyết định riêng việc giữ hay hủy dữ liệu chưa sync khi revoke; mặc định bảo mật ưu tiên xóa local theo tài liệu nghiệp vụ.

## 7. Feature flag model

Scope:

- Global: SuperAdmin quản lý, `TenantId = NULL`.
- Tenant: TenantAdmin override global.
- Default: giá trị an toàn trong code nếu chưa có setting.

Ưu tiên: `Tenant override > Global > Application default`.

Các flag ban đầu:

| Key | Type | Default | Server enforcement |
|---|---|---:|---|
| `RequireCustomerPhone` | bool | false | Chặn sync invoice thiếu phone |
| `EnableInventory` | bool | false | Ẩn module và chặn endpoint kho |
| `EnableAttendance` | bool | false | Chặn module chấm công |
| `EnableBluetoothPrint` | bool | false | Cho phép cấu hình máy in |
| `AutoRoundMoney` | bool | false | Server tính rounding theo rule |
| `RequireAdminPinForReverse` | bool | true | Bắt PIN khi Staff reverse |

UI chỉ ẩn/disable trải nghiệm; mọi rule quan trọng phải kiểm tra lại tại API.

## 8. Feature flag contract

```json
{
  "key": "RequireCustomerPhone",
  "value": true,
  "scope": "Tenant",
  "version": 12,
  "updatedAtUtc": "2026-10-08T09:00:00Z"
}
```

Validate type theo registry. Không cho phép JSON tùy ý biến thành quyền. Flag thay đổi phải tạo audit và event để client refresh cache.

## 9. API security baseline

- HTTPS bắt buộc qua Nginx.
- CORS allowlist, không dùng `*` với credential.
- Rate limit login, refresh, reverse và sync.
- Body size limit cho invoice batch.
- Validate model, enum và decimal range ở server.
- SQL injection phòng bằng EF parameterization.
- Security headers tại Nginx.
- Error response không lộ stack trace/database details.
- Dependency/container scan trong CI.
- Backup và log có kiểm soát truy cập.

## 10. Audit bắt buộc

Audit tối thiểu cho: login failure, role change, tenant lock, device revoke, price bulk update, feature flag update, reverse/correction invoice, sync rejection và force update.

Mỗi audit record nên có `Id`, `TenantId`, `ActorUserId`, `DeviceId`, `Action`, `EntityType`, `EntityId`, `BeforeJson`/`AfterJson` phù hợp, `CorrelationId`, `IpHash`, `CreatedAtUtc`.

Không lưu raw password, PIN hoặc token trong Before/After JSON.

## 11. Security test checklist

- Tenant A không thể đọc/sửa tenant B bằng body, query hoặc URL.
- JWT hết hạn/revoked bị từ chối.
- Staff không sửa BasePrice/bulk update.
- Staff không reverse nếu PIN sai hoặc thiếu.
- Feature flag server enforcement hoạt động khi client giả mạo UI.
- Payload sync thay đổi sau khi ký/hash bị từ chối.
- SQL injection, oversized body, brute force login và replay refresh token được kiểm tra.
