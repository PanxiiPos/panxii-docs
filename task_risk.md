# Task va Risk Register

## Muc tieu

Theo doi cac rui ro ky thuat da phat hien trong backend va chuyen chung thanh task co owner, tieu chi nghiem thu va test bat buoc.

Trang thai dung trong file:

- `Open`: chua bat dau.
- `In Progress`: dang trien khai tren feature branch.
- `Blocked`: bi chan boi dependency/quyet dinh.
- `Done`: da test va merge vao `main`.

## Risk register

| ID | Muc do | Rui ro | Anh huong | Xu ly bat buoc | Trang thai |
|---|---|---|---|---|---|
| RISK-001 | High | `TenantContext` mutable dang dung Singleton co the lam tenant cua request nay anh huong request khac | Cross-tenant data leak | Doi lifetime sang Scoped; set tenant tu authenticated claims trong middleware; khong set tenant tu header client | Open |
| RISK-002 | High | API chua co authentication/authorization va tenant middleware | Endpoint co the bi goi khong xac thuc, tenant isolation khong duoc bao ve trong production | Them JWT authentication, policy/role authorization, tenant resolution tu claim va `[Authorize]` cho endpoint business | Open |
| RISK-003 | Medium | Validation dang nem exception o service va controller tu format error | Error contract khong dong nhat, kho mo rong va kho quan sat | Doi sang `Result<T>`/error catalog va global error middleware | Open |
| RISK-004 | Medium | Docker local co the dung volume cu sau khi sua migration | Schema local khong dong bo voi migration | Huong dan reset dev volume co chu y va verify `__EFMigrationsHistory` | Open |

## Task backlog

### TASK-001 - Scoped TenantContext va tenant middleware

**Risk lien quan:** `RISK-001`  
**Priority:** P0  
**Branch goi y:** `feature/tenant-context-middleware`

#### Pham vi

- Doi `TenantContext` tu `Singleton` sang `Scoped`.
- Them `ITenantResolver`/tenant middleware trong Application interface va Infrastructure/API adapter phu hop.
- Lay `TenantId` tu JWT claim da xac thuc; khong tin `X-Tenant-Id` tu client.
- Tu choi request neu claim thieu, tenant khong ton tai, tenant bi suspend hoac claim khong hop le.
- Dam bao `AppDbContext` dung tenant context cua cung request.

#### Tieu chi nghiem thu

- Hai request dong thoi cua hai tenant khong the doc du lieu cua nhau.
- Global query filter dung tenant tu authenticated claim.
- Khong con state tenant mutable dung chung giua request.
- Co integration test cross-tenant read/write va request khong co tenant claim.
- `TenantContext` lifetime la Scoped trong DI.

#### Test bat buoc

- `TenantIsolationTests`.
- Request tenant A khong thay product/invoice cua tenant B.
- Concurrent requests voi tenant A/B khong lam leak context.
- Tenant revoked/suspended tra error code dung contract.

### TASK-002 - JWT authentication va authorization boundary

**Risk lien quan:** `RISK-002`  
**Priority:** P0  
**Branch goi y:** `feature/api-auth-authorization`

#### Pham vi

- Them `AddAuthentication().AddJwtBearer(...)` va `UseAuthentication()` truoc `UseAuthorization()`.
- Tao policy cho `Staff`, `TenantAdmin`, `SuperAdmin`.
- Them `[Authorize]` va policy cho products, customers, invoices va sync endpoints.
- Map role/tenant/device claims thanh application context qua interface.
- Khong log token, password, refresh token hoac SQLCipher key.
- Thay tenant development hardcode bang signed claims khi chay ngoai local Development.

#### Tieu chi nghiem thu

- Anonymous request vao business endpoint tra `401`.
- User dung role sai tra `403`.
- Token tenant A khong doc/ghi du lieu tenant B.
- Token het han, invalid signature va device revoked duoc map dung error contract.
- Swagger co Bearer security scheme cho endpoint protected.

#### Test bat buoc

- Authentication integration tests.
- RBAC matrix tests cho Staff/TenantAdmin/SuperAdmin.
- Invalid/expired/revoked token tests.
- Cross-tenant authorization tests.

### TASK-003 - Global Result va error middleware

**Risk lien quan:** `RISK-003`  
**Priority:** P1  
**Branch goi y:** `feature/global-error-contract`

#### Pham vi

- Service/handler tra `Result<T>` cho validation va business rejection.
- Global middleware map `Error.Code`, `Args`, `FieldErrors`, `Retryable` thanh API envelope.
- Controller khong catch exception nghiep vu rieng le.
- Exception chi danh cho infrastructure/unexpected failure va khong lo stack trace ra client.

#### Tieu chi nghiem thu

- Paging sai tra error code on dinh.
- Tat ca controller dung cung error envelope.
- Error code co trong catalog va co mapping i18n key.
- Co test middleware cho 400/401/403/404/409/422/500.

### TASK-004 - Charset va collation cho tieng Viet co dau

**Risk lien quan:** `RISK-005`  
**Priority:** P1  
**Branch goi y:** `feature/database-vietnamese-charset`

#### Van de can giai quyet

Database phai luu va tim kiem dung tieng Viet co dau, bao gom `Nguyen`, `Nguyễn`, `Đồ uống`, `Cà phê` va cac ky tu Unicode khac. Khong dung `latin1` hoac collation khong tuong thich Unicode.

#### Pham vi

- Dat database default `CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci` cho MySQL 8.
- Dat table/string columns quan trong ve `utf8mb4`; EF Core mapping khong ep ve ASCII ngoai UUID/code ky thuat.
- Dat connection option/collation ro rang neu provider can.
- Seed data development phai co du lieu tieng Viet co dau.
- Kiem tra migration SQL, schema, connection va API JSON encoding deu la UTF-8.
- Khong doi charset cua cot ma khong co migration/back-up/rollback plan.

#### Tieu chi nghiem thu

- `SHOW VARIABLES LIKE 'character_set%';` cho thay `utf8mb4` o database/connection.
- `SHOW VARIABLES LIKE 'collation%';` dung collation da chot.
- Insert/retrieve `Nguyễn Khách Hàng`, `Đồ uống`, `Cà phê sữa` khong bi loi hoac bien dang.
- Search API tim thay tieng Viet co dau va khong mat dau trong JSON response.
- Migration idempotent va khong lam mat du lieu Unicode hien co.
- Workbench hien thi dung ky tu co dau.

#### Test bat buoc

- Database integration test insert/read/update Unicode.
- API contract test voi request/response tieng Viet.
- Search test co dau va khong dau theo business rule da chot.
- Restore/backup test khong lam thay doi charset/collation.

#### SQL kiem tra local

```sql
SELECT @@character_set_database, @@collation_database;
SHOW CREATE TABLE tenants;
SHOW CREATE TABLE products;
```

## Definition of Done chung

- Task co branch rieng va commit theo [commit_convention.md](commit_convention.md).
- Co test phu hop voi risk level.
- Co migration/rollback plan neu cham database.
- Co cap nhat API/OpenAPI neu thay doi contract.
- Co review va test local truoc khi merge `main`.
- Cap nhat trang thai task trong file nay sau khi merge.
