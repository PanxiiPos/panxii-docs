# RISK-BASED TESTING SPECIFICATION

**Phạm vi:** POS SaaS multi-tenant, offline-first, API v1, MySQL/EF Core, SQLite/SQLCipher, RabbitMQ và SignalR  
**Mục tiêu:** chứng minh 4 rủi ro ưu tiên cao được kiểm soát bằng test có thể lặp lại, có dữ liệu đầu vào, tiêu chí pass/fail và bằng chứng.

---

## 1. Ma trận rủi ro và chiến lược kiểm thử

| Risk ID | Nhóm rủi ro | Xác suất | Tác động | Mức ưu tiên | Chiến lược chính |
|---|---|---:|---:|---:|---|
| R-001 | Cross-Tenant Data Leak | Thấp | Cực kỳ nghiêm trọng | P0 | Security integration, authorization, property-based tenant tests, automated CI |
| R-002 | Lệch giờ / sai thứ tự sync | Cao | Nghiêm trọng | P0 | Clock injection, ordering/convergence, retry/late-message tests |
| R-003 | SQLite hỏng / máy in lỗi | Rất cao | Trung bình | P1 | Fault injection, crash/restart, SQLCipher recovery, out-of-band print tests |
| R-004 | API breaking change | Trung bình | Nghiêm trọng | P0 | OpenAPI lint/diff, consumer contract, expand/contract compatibility tests |

### 1.1. Quy tắc mức độ

- **P0:** không được release nếu còn một test fail hoặc chưa có bằng chứng kiểm thử.
- **P1:** được release chỉ khi có workaround rõ, dữ liệu không mất và issue được chấp thuận.
- Mọi test bảo mật phải chạy với ít nhất hai tenant độc lập.
- Không dùng dữ liệu khách hàng thật trong test.

---

## 2. Môi trường kiểm thử chuẩn

### 2.1. Thành phần

```text
Test runner
  -> ASP.NET API test host
  -> MySQL 8 isolated container
  -> RabbitMQ isolated vhost/queue
  -> SQLite/SQLCipher test database
  -> Fake SignalR client
  -> Fake printer adapter
```

### 2.2. Test doubles bắt buộc

| Dependency | Test double |
|---|---|
| Current time | `FakeClock`/`IClock` |
| Tenant/user/device | `TestTenantContext` |
| RabbitMQ | Test broker hoặc in-memory publisher có ack/nack |
| SignalR | Fake hub recorder |
| Printer | `IPrinter` fake có thể timeout/fail |
| SQLite | Temporary SQLCipher file hoặc test database |
| External API | Stub server/HTTP mock |

Test không được dùng `DateTime.Now`, thời gian thực hoặc database dùng chung giữa test.

### 2.3. Test data tối thiểu

- Tenant A và Tenant B.
- Mỗi tenant có một Staff, một TenantAdmin và một device.
- Product trùng SKU giữa hai tenant nhưng khác giá/tên.
- Customer và invoice thuộc từng tenant.
- Invoice `Sale` đã `Synced`, invoice `Reverse`, invoice `Correction`.
- Hai cursor sync khác nhau và message đến đúng thứ tự/sai thứ tự.

---

## 3. R-001: Cross-Tenant Data Leak

### 3.1. Mục tiêu

Chứng minh user, device, query, command, event và cache của tenant A không thể đọc, sửa, sync hoặc nhận dữ liệu của tenant B, kể cả khi client giả mạo `TenantId`, resource ID hoặc routing key.

### 3.2. Controls cần xác minh

- `ITenantContext` lấy tenant từ JWT/server context.
- EF Core Global Query Filter áp dụng cho mọi entity tenant-aware.
- Interceptor/SaveChanges guard không cho insert/update sai tenant.
- Authorization kiểm tra role và tenant ownership.
- Cache key có tenant scope.
- RabbitMQ/SignalR group dùng `tenant:{TenantId}`.
- SuperAdmin path là ngoại lệ explicit, có audit.

### 3.3. Test cases

| ID | Loại | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-001 | Integration | Staff A gọi `GET /products` | Chỉ nhận product A |
| T-002 | Integration | Staff A gọi `GET /invoices/{invoiceB}` | 404/403 theo policy, không lộ dữ liệu B |
| T-003 | Security | A gửi body/query/header `tenantId = B` | Server bỏ qua, scope vẫn là A |
| T-004 | Security | Admin A PATCH product B bằng ID hợp lệ | Bị từ chối, DB B không đổi |
| T-005 | Database | Insert entity A với `TenantId = B` | Guard/interceptor reject hoặc ghi đúng A; không được ghi B |
| T-006 | Query filter | Bỏ tenant claim hoặc tenant context null | Không trả dữ liệu tenant |
| T-007 | Sync | Device A gửi invoice có product B | Reject với error code phù hợp |
| T-008 | SignalR | Invoice A được sync | Device B không nhận event; device A cùng tenant nhận |
| T-009 | Cache | Hai tenant request cùng resource key | Response không bị dùng chéo tenant |
| T-010 | SuperAdmin | SuperAdmin xem dữ liệu platform | Chỉ endpoint explicit, có audit, không mở bypass cho role khác |
| T-011 | Fuzz | Thay đổi resource ID, tenant header, query filter ngẫu nhiên | Không có response chứa dữ liệu chéo tenant |
| T-012 | Regression | Chạy toàn bộ endpoint contract với token A/B | Không endpoint nào trả entity ngoài scope |

### 3.4. Test tự động bắt buộc

- Tạo test fixture chạy cùng một test suite cho tenant A và B.
- So sánh mọi ID trong response với tập ID được phép của tenant hiện tại.
- Quét response JSON để phát hiện `tenantId` ngoài scope.
- Kiểm tra SQL/query log của các query quan trọng có tenant predicate hoặc được bảo vệ bởi filter.
- Chạy mutation test: bỏ `X-Tenant-Id`, đổi header, đổi JWT tenant claim trong test.

### 3.5. Pass/fail

**Pass khi:** mọi T-001 đến T-012 pass; không có PII hoặc entity B trong response/log/event của A; không có đường code production dùng `IgnoreQueryFilters()` ngoài platform path được phê duyệt.

**Fail ngay khi:** một request tenant A đọc được một field của B, cache/event chéo tenant, hoặc test phải tin `TenantId` từ request body.

---

## 4. R-002: Lệch giờ / sai thứ tự sync

### 4.1. Mục tiêu

Đảm bảo thời gian server là nguồn gốc ghi sổ, thứ tự thay đổi được xác định bằng cursor/sequence ổn định, và retry/late message không làm mất hoặc ghi đè dữ liệu sai.

### 4.2. Controls cần xác minh

- `CreatedAt`, `SyncedAt`, `ProcessedAt` server dùng UTC.
- Client timestamp chỉ là metadata nghiệp vụ, không quyết định thứ tự server.
- Cursor không chỉ dựa vào clock của mobile.
- Event có sequence/version hoặc ordering key ổn định.
- Upsert local chỉ nâng version, không ghi đè bản mới bằng bản cũ.
- Retry cùng operation id idempotent.
- Clock skew được đo và không làm thay đổi sổ giao dịch.

### 4.3. Test cases

| ID | Loại | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-101 | Unit | Client clock nhanh 2 giờ gửi invoice | Server `SyncedAt` làm timestamp ghi sổ |
| T-102 | Unit | Client clock chậm 24 giờ gửi invoice | Không làm lùi thứ tự server |
| T-103 | Integration | Hai invoice có client time ngược nhau | Thứ tự server dựa sequence/server time, không dựa client time |
| T-104 | Integration | Pull cursor nhận event 10 rồi 11 | Local apply 10 -> 11, cursor tăng đúng |
| T-105 | Integration | Event 11 đến trước event 10 | Không ghi đè version 11 bằng version 10; xử lý late event an toàn |
| T-106 | Integration | Cùng event được giao hai lần | Consumer deduplicate, state cuối không đổi |
| T-107 | E2E | API commit xong nhưng response timeout | Retry cùng idempotency key trả kết quả cũ |
| T-108 | Chaos | RabbitMQ delay 1-5 phút | Invoice server không mất; device cuối cùng converge |
| T-109 | Unit | DST/timezone thay đổi | UTC instant không đổi; UI chỉ đổi cách hiển thị |
| T-110 | Integration | Cursor hết hạn | Trả `ERR_CURSOR_EXPIRED`, full resync không tạo duplicate |
| T-111 | Concurrency | Hai worker xử lý cùng outbox | Lease/lock bảo đảm không làm sai thứ tự; consumer idempotent |
| T-112 | Property | Sinh chuỗi event trùng/trễ/ngẫu nhiên | State cuối bằng state từ chuỗi event hợp lệ theo version |

### 4.4. Tiêu chí ordering/convergence

Với cùng một tập command hợp lệ, sau khi tất cả message đã được xử lý:

```text
Server ledger = expected ledger
Local replica của mọi device active = Server ledger
LastServerCursor >= cursor của mọi change đã apply
Version local không giảm
SyncedAtUtc không lấy từ client timestamp
```

### 4.5. Pass/fail

**Pass khi:** T-101 đến T-112 pass; mọi invoice có timestamp server; retry và late message không tạo duplicate hoặc rollback state; hai thiết bị converge về cùng snapshot sau pull.

**Fail ngay khi:** client đổi clock làm thay đổi thứ tự ghi sổ, cursor bị bỏ qua, version giảm hoặc local replica giữ dữ liệu cũ sau khi đã nhận dữ liệu mới hơn.

---

## 5. R-003: SQLite hỏng / máy in lỗi

### 5.1. Mục tiêu

Đảm bảo lỗi local database, process crash, hết dung lượng, SQLCipher key lỗi hoặc printer timeout không làm sai trạng thái invoice và không chặn việc ghi nhận giao dịch theo policy.

### 5.2. Controls cần xác minh

- SQLite bật WAL theo policy và transaction được dùng cho invoice + outbox.
- SQLCipher key lưu secure storage, không log/commit.
- Local write hoàn tất trước khi UI xác nhận invoice.
- Sync worker có lease/retry, không mất Pending record sau process crash.
- Printing là side effect out-of-band, không nằm trong transaction ghi invoice.
- Print job có trạng thái riêng, idempotency và retry/manual reprint.
- Khi device revoke, database/key/token bị xóa theo policy.

### 5.3. Test cases SQLite/SQLCipher

| ID | Loại | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-201 | E2E | Tạo invoice offline | Invoice và outbox commit cùng nhau |
| T-202 | Crash | Kill process sau khi ghi invoice trước outbox | Không có invoice local ở trạng thái đã thanh toán nhưng thiếu outbox |
| T-203 | Crash | Kill process sau outbox commit trước UI ack | Mở app lại thấy invoice Pending |
| T-204 | Recovery | Thiết bị mất điện khi WAL đang ghi | Database mở lại được hoặc recovery báo lỗi rõ, không silent corruption |
| T-205 | Fault | Hết dung lượng disk khi commit | Transaction rollback, UI báo lỗi retryable; không tạo partial invoice |
| T-206 | Security | Mở file SQLite không có key | Không đọc được dữ liệu |
| T-207 | Security | SQLCipher key sai/hỏng | App vào recovery/force logout theo policy, không loop vô hạn |
| T-208 | Concurrency | Hai worker claim cùng Pending row | Chỉ một worker xử lý; không duplicate sync |
| T-209 | Recovery | Database local hỏng | App xóa/recreate DB theo policy, hiển thị trạng thái rõ; server invoice đã sync vẫn pull lại được |
| T-210 | Revoke | API trả `401 ERR_DEVICE_REVOKED` | Dừng worker, đóng DB, xóa file/key/token, về login |
| T-211 | Migration | Local schema cũ mở bằng app mới | Migration thành công hoặc yêu cầu resync có kiểm soát |
| T-212 | WAL | Nhiều read đồng thời một write | UI không block bất thường và transaction vẫn nhất quán |

### 5.4. Test cases máy in out-of-band

| ID | Loại | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-220 | Integration | Invoice commit, printer online | Invoice Synced; print job Completed |
| T-221 | Fault | Printer timeout sau invoice commit | Invoice vẫn Synced; print job Failed/Retryable |
| T-222 | Fault | Printer mất kết nối giữa chừng | Có thể retry/reprint, không tạo invoice thứ hai |
| T-223 | Retry | Worker retry print job | Không in duplicate ngoài policy; job có idempotency |
| T-224 | UX | User thanh toán khi máy in lỗi | UI báo invoice thành công và lỗi in riêng |
| T-225 | Security | Device không có quyền printer | Invoice không bị rollback; lỗi permission riêng |

### 5.5. Pass/fail

**Pass khi:** crash/fault không tạo partial invoice, invoice đã commit không bị rollback vì printer, Pending có thể tiếp tục sync, DB/key được bảo vệ và print failure tách khỏi payment state.

**Fail ngay khi:** máy in lỗi làm mất invoice, retry tạo duplicate invoice, SQLite xác nhận thanh toán khi transaction chưa commit, hoặc key/PII xuất hiện trong log.

---

## 6. R-004: API breaking change

### 6.1. Mục tiêu

Đảm bảo backend mới vẫn tương thích với mobile version được hỗ trợ và thay đổi breaking phải đi qua versioning/expand-contract.

### 6.2. Controls cần xác minh

- `openapi.yaml` là source contract máy đọc được.
- OpenAPI lint và breaking-change diff chạy trong CI.
- Success/error envelope không đổi ngoài version.
- Field mới được thêm optional trước khi bắt buộc.
- Không đổi kiểu/ý nghĩa/xóa field trong v1.
- Enum client có fallback `Unknown` hoặc server version hóa trước khi thêm giá trị.
- Database expand/contract tương thích với API/mobile cũ.

### 6.3. Phân loại thay đổi

| Thay đổi | v1 có breaking không? | Cách xử lý |
|---|---:|---|
| Thêm optional response field | Không | Add và test client cũ |
| Thêm optional request field | Không | Backend default rõ |
| Xóa response field | Có | Deprecate rồi v2 |
| Đổi field type | Có | Field mới hoặc v2 |
| Đổi enum hiện có | Có | Version/fallback |
| Đổi `data` envelope | Có | Không làm trong v1 |
| Đổi error code | Có | Giữ alias/deprecate, cập nhật i18n |
| Đổi decimal thành string/number | Có | ADR + version/dual field |
| Đổi required field | Có | Expand/contract |

### 6.4. Test cases

| ID | Loại | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-301 | CI | Lint OpenAPI | Spec hợp lệ, không warning nghiêm trọng |
| T-302 | CI | So sánh spec PR với baseline v1 | Breaking diff bị fail hoặc cần approval |
| T-303 | Contract | Mobile cũ gọi API mới | Parse được response cũ và không crash |
| T-304 | Contract | Mobile mới gọi API cũ | Chỉ dùng field được hỗ trợ; fallback đúng |
| T-305 | API | Backend thêm optional response field | Client cũ vẫn hoạt động |
| T-306 | API | Backend xóa/đổi field bắt buộc | CI reject nếu chưa tạo v2/compatibility plan |
| T-307 | Error | Error code mới xuất hiện | Có catalog + vi/en key trước release |
| T-308 | Enum | Server thêm enum mới | Mobile không crash, hiển thị Unknown/fallback |
| T-309 | Migration | DB expand migration trong lúc app cũ chạy | App cũ và mới cùng hoạt động |
| T-310 | Rollback | Rollback API image sau migration | Schema vẫn backward-compatible |
| T-311 | Sync | PayloadVersion cũ gửi lên server mới | Server xử lý hoặc trả `ERR_UNSUPPORTED_PAYLOAD_VERSION` chuẩn |
| T-312 | Generate | Generate client từ OpenAPI | Model/serialization compile và contract test pass |

### 6.5. Pass/fail

**Pass khi:** OpenAPI lint, diff, consumer contract và compatibility matrix pass; client cũ không crash; mọi breaking change có v2 hoặc expand/contract plan được phê duyệt.

**Fail ngay khi:** đổi envelope/error code/field type làm mobile không parse được, hoặc migration xóa/đổi cột khiến phiên bản app đang hỗ trợ không hoạt động.

---

## 7. Tự động hóa CI/CD

### Pull request gate

```text
1. OpenAPI lint
2. OpenAPI breaking-change diff
3. Unit tests
4. Tenant isolation/security integration tests
5. Ordering/convergence tests với FakeClock
6. SQLite crash/recovery tests
7. Printer adapter fault tests
8. API consumer contract tests
9. Migration validation
```

### Staging gate

```text
1. MySQL/RabbitMQ isolated integration
2. Offline E2E với process kill/network toggle
3. Clock skew và message delay chaos
4. SQLCipher recovery test
5. Printer timeout/reprint test
6. Mobile old/new compatibility test
7. Backup restore smoke test
```

### Release gate

- P0 không còn test fail.
- Không có Cross-Tenant leak finding mở.
- Có report ordering/convergence.
- Có bằng chứng SQLite recovery và print out-of-band.
- Có OpenAPI diff report.
- Có traceability từ risk ID đến test case và release artifact.

---

## 8. Traceability matrix

| Risk | Controls | Test cases | Evidence |
|---|---|---|---|
| R-001 | Tenant context, query filter, interceptor, auto test | T-001..T-012 | CI report, security test log, audit sample |
| R-002 | Server timestamp, cursor/version, idempotency | T-101..T-112 | Ordering report, convergence hash, message trace |
| R-003 | SQLCipher, WAL, transaction, print adapter | T-201..T-225 | Recovery report, printer fault report |
| R-004 | OpenAPI, expand/contract, versioning | T-301..T-312 | OpenAPI diff, consumer contract report |

## 9. Exit criteria

Một release chỉ đạt khi:

1. Tất cả test P0 pass trên CI và staging.
2. Không có leak cross-tenant hoặc dữ liệu giao dịch duplicate.
3. Server ledger và local replica converge sau sync/retry.
4. Printer failure không ảnh hưởng invoice state.
5. OpenAPI không có breaking change ngoài version/approval.
6. Test artifact được lưu cùng release và liên kết với commit/build SHA.
7. Có rollback và incident runbook phù hợp.
