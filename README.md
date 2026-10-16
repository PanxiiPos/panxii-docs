# POS SaaS Documentation Hub

## Dự án

Đây là bộ tài liệu của **Panxii**, nền tảng POS SaaS multi-tenant và offline-first. Xem [project_overview.md](project_overview.md) để đọc phần giới thiệu sản phẩm, phạm vi và kiến trúc tổng quan.

Bộ tài liệu thiết kế và phát triển hệ thống POS SaaS theo định hướng:

- Multi-tenant SaaS
- Offline-first
- Flutter + BLoC
- ASP.NET Core 8 Clean Architecture
- MySQL 8 + EF Core 8
- RabbitMQ + SignalR
- i18n với `easy_localization`
- UI dùng `flex_color_scheme`, `hugeicons`, `forui`

## 1. Đọc nhanh theo thứ tự

Khi bắt đầu một feature mới, đọc theo thứ tự sau:

```text
1. business_rule.md
2. SDLC&Architecture.md
3. architecture/system_architecture.md
4. architecture/domain_use_cases.md
5. architecture/adr/README.md
6. database/database_design.md
7. api/api_contract.md
8. api/openapi.yaml
9. api/error_codes.md
10. api/api_compatibility_matrix.md
11. security/security_rbac_feature_flags.md
12. backend/backend_coding_standard.md hoặc frontend/frontend_coding_standard.md
13. sync/data_sync_flow_rabbitmq.md nếu có sync/realtime
14. testing/testing_strategy.md
15. testing/role_based_test_cases.md khi cần kiểm thử chức năng/RBAC theo role
16. testing/risk_based_testing_spec.md nếu chạm rủi ro P0/P1
17. infrastructure/infrastructure_setup.md
18. release/release_management.md
19. development_workflow.md
20. commit_convention.md
21. task_risk.md
```

Không cần đọc toàn bộ tài liệu cho mọi thay đổi. Chọn nhánh backend/frontend bên dưới, nhưng luôn bắt đầu bằng business rule và architecture.

---

## 2. Bản đồ tài liệu

### 2.1. Quy tắc gốc

| File | Vai trò |
|---|---|
| [business_rule.md](business_rule.md) | Quy tắc nghiệp vụ bất khả xâm phạm: RBAC, reverse invoice, offline sync, i18n, feature flags |
| [SDLC&Architecture.md](SDLC%26Architecture.md) | Master plan, technology stack, lifecycle phát triển và triển khai |
| [setup_env.md](setup_env.md) | Cài SDK, khởi tạo source, package Flutter, Docker local và môi trường |
| [commit_convention.md](commit_convention.md) | Quy ước Conventional Commits, scope, breaking change, PR và checklist trước commit |
| [development_workflow.md](development_workflow.md) | Quy trình tạo branch, Pull Request, test, merge và bảo vệ `main` |
| [task_risk.md](task_risk.md) | Task/risk register cho tenant context, authentication, error contract và Unicode database |

### 2.2. Architecture

| File | Vai trò |
|---|---|
| [system_architecture.md](architecture/system_architecture.md) | Sơ đồ tổng thể, layer, dependency direction và data flow |
| [domain_use_cases.md](architecture/domain_use_cases.md) | Actor, aggregate, state machine, use case catalog và invariant |
| [adr/README.md](architecture/adr/README.md) | Nơi ghi các quyết định kiến trúc có ảnh hưởng lâu dài |

### 2.3. Database và persistence

| File | Vai trò |
|---|---|
| [database_design.md](database/database_design.md) | Schema MySQL/SQLite, entity, index, tenant isolation, invoice immutable, outbox |
| [migration_seed_strategy.md](backend/migration_seed_strategy.md) | EF Core migration, seed, expand/contract và rollback data |

> `database/` là nhóm tài liệu database và persistence.

### 2.4. API và integration contract

| File | Vai trò |
|---|---|
| [api_contract.md](api/api_contract.md) | Quy tắc request/response, envelope, DTO, paging, idempotency và endpoint |
| [openapi.yaml](api/openapi.yaml) | Contract máy đọc được cho Swagger, generator và contract test |
| [error_codes.md](api/error_codes.md) | Catalog error code, HTTP status, retry policy và i18n key |
| [api_compatibility_matrix.md](api/api_compatibility_matrix.md) | Compatibility giữa mobile/API, versioning và breaking-change gate |
| [data_sync_flow_rabbitmq.md](sync/data_sync_flow_rabbitmq.md) | Offline-to-online, RabbitMQ, transactional outbox, retry/DLQ và SignalR pull |

### 2.5. Security

| File | Vai trò |
|---|---|
| [security_rbac_feature_flags.md](security/security_rbac_feature_flags.md) | JWT, RBAC, tenant isolation, kill-switch, secrets, audit và feature flags |
| [privacy_data_retention.md](security/privacy_data_retention.md) | Phân loại PII, retention, export, anonymization và incident privacy |

### 2.6. Coding standards

| File | Vai trò |
|---|---|
| [backend_coding_standard.md](backend/backend_coding_standard.md) | Clean Architecture, ownership, reuse, dependency, Result, EF Core, anti-pattern |
| [frontend_coding_standard.md](frontend/frontend_coding_standard.md) | Flutter/BLoC, offline UI, API parsing, i18n, `flex_color_scheme`, `hugeicons`, `forui` |
| [design_system.md](frontend/design_system.md) | Color, typography, spacing, component states và accessibility |
| [mobile_screen_flow_api_mapping.md](frontend/mobile_screen_flow_api_mapping.md) | Screen descriptions, navigation flow, API request/response mapping và contract gaps cho Flutter mobile |

### 2.7. Testing

| File | Vai trò |
|---|---|
| [testing_strategy.md](testing/testing_strategy.md) | Unit, integration, API contract, offline E2E, RabbitMQ chaos và k6 |
| [role_based_test_cases.md](testing/role_based_test_cases.md) | Test case chức năng theo role, boundary, validation và tenant isolation |
| [risk_based_testing_spec.md](testing/risk_based_testing_spec.md) | Test spec cho cross-tenant leak, lệch giờ/sai thứ tự sync, SQLite/printer failure và API breaking change |

### 2.8. Infrastructure và release

| File | Vai trò |
|---|---|
| [infrastructure_setup.md](infrastructure/infrastructure_setup.md) | Docker, Nginx, VPS, MySQL, RabbitMQ, CI/CD, health check |
| [operations_runbook.md](infrastructure/operations_runbook.md) | Xử lý outbox, DLQ, MySQL, SignalR, revoke và incident |
| [disaster_recovery.md](infrastructure/disaster_recovery.md) | Backup, restore, RPO/RTO, mất VPS, lộ secret |
| [slo_observability.md](infrastructure/slo_observability.md) | SLO, metrics, alert threshold, error budget và correlation |
| [release_management.md](release/release_management.md) | Versioning, rollout, migration gate, mobile compatibility và rollback |

### 2.9. Module nghiệp vụ mở rộng

| File | Vai trò |
|---|---|
| [payment_specification.md](payment/payment_specification.md) | Payment model, cash/non-cash, capture/refund và reconciliation |
| [sqlite_print_recovery_spec.md](sync/sqlite_print_recovery_spec.md) | SQLite/SQLCipher recovery và print out-of-band |

---

## 3. Trình tự check khi phát triển feature

### Bước 1: Phân tích nghiệp vụ

Đọc:

1. [business_rule.md](business_rule.md)
2. [domain_use_cases.md](architecture/domain_use_cases.md)
3. [security_rbac_feature_flags.md](security/security_rbac_feature_flags.md)

Chốt:

- Actor nào được dùng feature?
- Tenant scope là gì?
- Có thay đổi dữ liệu invoice đã sync không?
- Có cần reverse entry không?
- Có phụ thuộc feature flag không?
- Có cần hoạt động offline không?
- Error code nào cần thêm?

### Bước 2: Kiểm tra kiến trúc

Đọc:

1. [system_architecture.md](architecture/system_architecture.md)
2. [backend_coding_standard.md](backend/backend_coding_standard.md) hoặc [frontend_coding_standard.md](frontend/frontend_coding_standard.md)
3. [architecture/adr/README.md](architecture/adr/README.md)

Chốt:

- Feature thuộc module nào?
- Dependency đi theo hướng nào?
- Owner của business rule là class/service nào?
- Có cần tạo ADR không?
- Có đang tạo abstraction hoặc code lặp không cần thiết không?

### Bước 3: Kiểm tra data model

Đọc [database_design.md](database/database_design.md) và [migration_seed_strategy.md](backend/migration_seed_strategy.md).

Chốt:

- Có entity/table mới không?
- Có `TenantId`, audit, `ExtraDataJson` không?
- Có index/unique key/FK cần thêm không?
- Có ảnh hưởng invoice immutable không?
- Migration có backward-compatible với mobile cũ không?
- Có seed/reference data không?

### Bước 4: Thiết kế API

Đọc:

1. [api_contract.md](api/api_contract.md)
2. [openapi.yaml](api/openapi.yaml)
3. [error_codes.md](api/error_codes.md)
4. [api_compatibility_matrix.md](api/api_compatibility_matrix.md)

Chốt:

- Request DTO và response DTO cụ thể.
- Success dùng `data` + `meta`.
- Error dùng `data: null` + `error.code` + `args` + `fieldErrors`.
- Command retry có `Idempotency-Key`.
- Có `X-Correlation-Id`.
- OpenAPI schema và operationId được cập nhật.
- Mobile có mapping `vi.json`/`en.json`.

### Bước 5: Thiết kế sync nếu có offline/realtime

Đọc [data_sync_flow_rabbitmq.md](sync/data_sync_flow_rabbitmq.md) và [sqlite_print_recovery_spec.md](sync/sqlite_print_recovery_spec.md) nếu có local DB/printer.

Chốt:

- Local table/outbox nào thay đổi?
- Operation ID là gì?
- Retry có idempotent không?
- Có transactional outbox không?
- Event RabbitMQ routing key nào?
- SignalR chỉ báo hiệu và client pull bằng cursor thế nào?
- Khi cursor hết hạn hoặc device bị revoke xử lý ra sao?

### Bước 6: Implement backend hoặc frontend

#### Backend

1. Domain entity/value object/rule.
2. Application command/query, validator, handler.
3. Repository port và infrastructure adapter.
4. Transaction/outbox.
5. API controller mỏng.
6. OpenAPI và error mapping.

Check theo [backend_coding_standard.md](backend/backend_coding_standard.md).

#### Frontend

1. API DTO và local model.
2. Repository contract/implementation.
3. BLoC/Cubit và state.
4. SQLite/offline behavior.
5. UI bằng theme token.
6. Icon bằng `hugeicons`.
7. Dropdown/Dialog/DateRangePicker/Badge/Toast bằng `forui`.
8. Text bằng `easy_localization`.

Check theo [frontend_coding_standard.md](frontend/frontend_coding_standard.md) và [design_system.md](frontend/design_system.md).

### Bước 7: Kiểm thử

Đọc [testing_strategy.md](testing/testing_strategy.md), [role_based_test_cases.md](testing/role_based_test_cases.md) khi feature có quyền/chức năng theo role, và [risk_based_testing_spec.md](testing/risk_based_testing_spec.md) khi chạm rủi ro P0/P1.

Tối thiểu phải test:

- Tenant isolation.
- RBAC.
- Feature flag server-side.
- Error code và API envelope.
- Business invariant.
- Retry/idempotency.
- Offline restart/sync.
- Reverse invoice nếu liên quan.
- UI loading/empty/error/i18n nếu liên quan frontend.
- Risk test tương ứng trong [risk_based_testing_spec.md](testing/risk_based_testing_spec.md) nếu feature chạm tenant, sync, local DB/printer hoặc API contract.

### Bước 8: Migration, deploy và release

Đọc theo thứ tự:

1. [infrastructure_setup.md](infrastructure/infrastructure_setup.md)
2. [operations_runbook.md](infrastructure/operations_runbook.md)
3. [disaster_recovery.md](infrastructure/disaster_recovery.md)
4. [slo_observability.md](infrastructure/slo_observability.md)
5. [release_management.md](release/release_management.md)

Trước production phải có:

- Backup thành công.
- Migration SQL đã review.
- Image pin theo git SHA.
- Health check pass.
- Smoke test login/sync/pull.
- Rollback code/data plan.
- Monitoring và alert.

---

## 4. Nguyên tắc ưu tiên khi tài liệu mâu thuẫn

Khi phát hiện mâu thuẫn, ưu tiên kiểm tra theo thứ tự:

1. Business rule đã được phê duyệt.
2. ADR accepted gần nhất.
3. API/OpenAPI contract đang phát hành.
4. Database schema/migration hiện tại.
5. Coding standard.
6. Ví dụ trong tài liệu cũ.

Không tự chọn một cách im lặng. Ghi lại quyết định trong ADR và cập nhật các tài liệu liên quan.

## 5. Trạng thái triển khai hiện tại

### Backend

- ASP.NET Core 8 Clean Architecture đã có base code Domain, Application, Infrastructure và WebApi.
- OpenAPI/Swagger v1 đã cấu hình:
	- UI: `http://localhost:5099/swagger`
	- JSON: `http://localhost:5099/swagger/v1/swagger.json`
- API health check: `GET http://localhost:5099/api/v1/health`; deployment probes use `/health/live` and `/health/ready` (MySQL + RabbitMQ).
- Auth API: `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`; login body chỉ cần username/password, không cần nhập tenant hay device metadata.
- JWT tenant user có tenant/role/device/session claims; JWT SuperAdmin không có `tenant_id` và session lưu riêng trong `platform_admin_sessions`. Role policies bảo vệ Products, logout/revoke được kiểm tra server-side trên mỗi request, login/refresh rate limit 10 request/phút/IP.
- Backend hiện triển khai API SuperAdmin CRUD/lifecycle tenant, tạo tenant cùng TenantAdmin đầu tiên trong một transaction, quản lý các TenantAdmin tiếp theo và đọc/cập nhật global feature flags. Tenant soft-delete giữ nguyên dữ liệu nghiệp vụ và revoke sessions.
- Backend có API Staff đọc/tìm product/customer, đọc invoice/effective settings và `GET /api/v1/sync/pull?afterSequence=`. `POST /api/v1/invoices/sync`, TenantAdmin `POST /api/v1/products` và `POST /api/v1/customers` trả `202 Queued` sau RabbitMQ publisher confirm.
- Worker xử lý idempotent theo tenant/operation/payload hash; entity, operation, `server_changes` sequence và outbox commit cùng transaction. Sau commit, outbox phát full snapshot qua RabbitMQ; SignalR `/hubs/pos` chỉ join group theo tenant claim và gửi `DataCreated` tới client cùng tenant. Client insert snapshot lên đầu và deduplicate bằng `serverSequence`/`messageId`; `/sync/pull` khôi phục event bị lỡ. Mobile repository hiện chưa có Dart implementation để xác nhận UI listener/list insertion.
- TenantAdmin quản lý tài khoản Staff, vòng đời sản phẩm/customer, cấu hình cửa hàng và các thao tác invoice reverse/correction chưa được xác nhận triển khai.
- Access token mặc định 15 phút; refresh token opaque được rotate và chỉ lưu hash SHA-256; password lưu BCrypt hash.
- Middleware API đã chuẩn hóa correlation ID (`X-Correlation-Id`), exception/error envelope và localization theo `Accept-Language` (`vi`/`en`, mặc định `vi`); response có `Content-Language`.
- MVC result filter bọc response success/error theo `data` + `meta` hoặc error envelope; backend trả error code, mobile vẫn dịch qua `easy_localization`.
- Products query API: `GET http://localhost:5099/api/v1/products?search=&page=1&pageSize=50`.
- Products API trả `data` + `meta.correlationId` + `meta.paging`, hỗ trợ tìm theo tên/SKU/barcode và tenant filter.
- Clean Architecture flow của Products API: `Controller -> IProductService -> IUnitOfWork -> IProductRepository -> GenericRepository -> AppDbContext`.
- `IGenericRepository<TEntity>` cung cấp CRUD persistence dùng lại qua interface; repository chuyên biệt thêm query theo aggregate.
- `IUnitOfWork` chứa các repository interfaces và `EfUnitOfWork` giữ transaction boundary; controller/service không gọi trực tiếp `DbContext`.
- Interface được tổ chức theo layer: `Application/Interfaces/Services` chứa service contracts; `Application/Interfaces/Repositories` chứa `IGenericRepository`, repository contracts và `IUnitOfWork`.
- Controller chỉ phụ thuộc `IProductService`; service chỉ phụ thuộc `IUnitOfWork`; UoW expose repository interfaces; repository kế thừa generic repository và Infrastructure là layer duy nhất truy cập `AppDbContext`.
- Local environment chạy bằng `Development` qua file `.env` trong backend.

### Database local

- MySQL 8 chạy bằng Docker container `panxii_mysql_dev`.
- Connection local: `127.0.0.1:3306`.
- Database: `PanxiiDev`.
- User: `panxii_app`.
- EF Core migrations đã apply:
	- `InitialPosSchema`.
	- `AddDatabaseForeignKeys`.
- Các migrations đã apply: `InitialPosSchema`, `AddDatabaseForeignKeys`, `AddUserDevices`, `AddUserDeviceRefreshTokenHashIndex`, `SeparatePlatformAdmin`, `AddPlatformAdministration` và `AddTenantVersion`.
- Nhánh backend `feature/async-pos-realtime` bổ sung migrations `AddInvoiceSkuSnapshot`, `AddInvoiceLineAndTaxAmounts` và `AddRealtimeChangeFeed` (bao gồm operation ID); apply trước khi chạy API/Worker.
- Session SuperAdmin và tenant-user được tách riêng; `app_settings`, `platform_audit_events`, optimistic concurrency version và durable `server_changes` sequence đã có trong model/migration.
- Schema gồm `tenants`, `users`, `user_devices`, `platform_admins`, `platform_admin_sessions`, `app_settings`, `platform_audit_events`, `categories`, `products`, `customers`, `invoices`, `invoice_details`, `sync_operations`, `outbox_events` và `server_changes`.
- Các quan hệ nghiệp vụ dùng composite foreign key theo `(TenantId, Id)` để bảo vệ tenant isolation.
- Development seed script: `backend/database/seed-development.sql`.
- Tài khoản development (chỉ dùng local; seed reset mật khẩu theo role): `superadmin` / `SuperAdmin123`, `demo.admin` / `TenantAdmin123`, `demo.staff` / `Staff123`.
- Dữ liệu development: 1 tenant (`DEMO`), 2 tenant users, 1 platform admin, 2 categories, 2 products, 1 customer, 1 invoice, 2 invoice details, 1 sync operation và 1 outbox event.

Khởi động database local:

```powershell
cd backend
docker compose -f docker-compose.local.yml up -d --wait panxii-db-local rabbitmq
dotnet ef database update `
	--project Panxii.Infrastructure/Panxii.Infrastructure.csproj `
	--startup-project Panxii.WebApi/Panxii.WebApi.csproj

# Seed data development
$sql = Get-Content -Raw database/seed-development.sql
$sql | docker exec -i panxii_mysql_dev mysql `
	--host=127.0.0.1 --user=panxii_app --password=panxii_local_only --database=PanxiiDev

# Start API and command/outbox Worker after migrations and seed
docker compose -f docker-compose.local.yml up -d --build panxii-api panxii-worker
```

Swagger: `http://localhost:5099/swagger`. RabbitMQ management local: `http://localhost:15672`.
Nếu chạy API trực tiếp trên host thay vì Compose, dùng:

```powershell
dotnet run --project backend/Panxii.WebApi/Panxii.WebApi.csproj `
	--urls http://localhost:5099
```

### Quy tắc cập nhật README

`Document/README.md` là README tổng hợp duy nhất của dự án. Khi có thay đổi kiến trúc, database, API, môi trường hoặc workflow, cập nhật file này; không cập nhật các README riêng trong `backend/`, `mobile/` hoặc repository khác nếu không có yêu cầu riêng.

## 6. Tài liệu cần bổ sung khi bật module mới

Các module sau chưa phải core hiện tại; chỉ tạo tài liệu khi bắt đầu triển khai:

- Inventory specification.
- Attendance specification.
- Notification/email/SMS specification.
