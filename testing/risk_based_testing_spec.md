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
| T-008 | SignalR | Invoice A được tạo qua queue | Tenant A clients receive the full committed snapshot and sequence; tenant B receives nothing; event is emitted only after DB commit |
| T-009 | Cache | Hai tenant request cùng resource key | Response không bị dùng chéo tenant |
| T-010 | SuperAdmin | SuperAdmin xem dữ liệu platform | Chỉ endpoint explicit, có audit, không mở bypass cho role khác |
| T-011 | Fuzz | Thay đổi resource ID, tenant header, query filter ngẫu nhiên | Không có response chứa dữ liệu chéo tenant |
| T-012 | Regression | Chạy toàn bộ endpoint contract với token A/B | Không endpoint nào trả entity ngoài scope |
xem thêm tại [role_based_test_cases.md](testing/role_based_test_cases.md)

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
| T-104 | Integration | Pull `afterSequence` nhận change 10 rồi 11 | Local apply 10 -> 11; chỉ advance `nextSequence` after transaction commit |
| T-105 | Integration | Event 11 đến trước event 10 | Không ghi đè version 11 bằng version 10; xử lý late event an toàn |
| T-106 | Integration | Cùng event được giao hai lần | Consumer deduplicate, state cuối không đổi |
| T-107 | E2E | API commit xong nhưng response timeout | Retry cùng idempotency key trả kết quả cũ |
| T-108 | Chaos | RabbitMQ delay 1-5 phút | Invoice server không mất; device cuối cùng converge |
| T-109 | Unit | DST/timezone thay đổi | UTC instant không đổi; UI chỉ đổi cách hiển thị |
| T-110 | Integration | Pull với `afterSequence < 0`, limit 0 hoặc limit >200 | Trả `400 ERR_INVALID_PAGING`; sequence valid không hết hạn while retained change rows exist |
| T-111 | Concurrency | Hai worker publish cùng outbox event | Có thể giao trùng; client deduplicate theo messageId/sequence, không giảm version hoặc mất change |
| T-112 | Property | Sinh chuỗi event trùng/trễ/ngẫu nhiên | State cuối bằng state từ chuỗi event hợp lệ theo version |
| T-113 | Integration | Client nhận `DataCreated` cho Product/Customer/Invoice | Snapshot được upsert/deduplicated theo `messageId`/`serverSequence` và chèn lên đầu; không cần gọi refresh endpoint |
| T-114 | Integration | API chấp nhận command nhưng worker commit xong trước khi process chết | Redelivery không tạo duplicate; outbox vẫn phát một logical event, duplicates được client deduplicate |

### 4.4. Tiêu chí ordering/convergence

Với cùng một tập command hợp lệ, sau khi tất cả message đã được xử lý:

```text
Server ledger = expected ledger
Local replica của mọi device active = Server ledger
LastServerSequence >= sequence của mọi change đã apply
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

## 10. Gate resilience trước triển khai

Chạy `backend/scripts/test-resilience.ps1` từ thư mục backend. Các test nằm trong
`backend/Panxii.Tests/Resilience`; dùng MySQL, RabbitMQ, API, Worker và Nginx thực,
không dùng database/vhost đang có dữ liệu thật. CLI test có thể gây SIGKILL,
thay đổi trigger và xóa volume **trong project test tự sinh**.

| Nhóm P0 | Fault/boundary | Assertion bắt buộc |
|---|---|---|
| Availability | Broker down trước confirm | HTTP 503 `ERR_SERVICE_UNAVAILABLE`; không có entity/operation/change/outbox; live vẫn 200, ready 503; retry cùng UUID thành công |
| Durable queue | Worker dừng, broker restart sau 202 | Chưa có invoice khi worker dừng; command còn trong queue; khởi động worker tạo đúng một invoice |
| Cold-start | Restart API/Worker ba lần; khởi tạo EF/auth/query chỉ đọc trước HTTP/subscribe | Không thêm invoice/detail/receipt/change/outbox lúc startup; mỗi hóa đơn mới đầu tiên có Completed receipt và snapshot trong 1500 ms; không hóa đơn warm-up/bỏ mẫu |
| Crash giữa chừng | Chặn outbox insert bằng MySQL named lock sau SaveChanges thứ nhất, rồi SIGKILL worker hoặc MySQL | Chứng minh worker đã đến boundary; không đọc được partial commit; rollback sau crash; redelivery commit đúng một lần |
| Atomicity | Trigger làm detail thứ hai hoặc outbox insert thất bại | Không còn invoice/detail/operation/change/outbox; bỏ fault rồi retry cùng operation thành công |
| Duplicate | 16 request cùng UUID; client không đọc response rồi reset socket sau DB commit | Một invoice, đúng số detail, một Completed operation, một change, một logical DataCreated outbox sau queue drain (kể cả unacked) |
| Payload mismatch | Cùng UUID nhưng timestamp payload khác | CommandRejected `ERR_SYNC_PAYLOAD_MISMATCH`; snapshot và timestamp gốc không đổi |
| Batch/reconnect | API ngừng sau 3/8 invoice; client time +2h/-24h; replay mọi UUID | Đủ 8 invoice, không duplicate; pull page size 2 strictly increasing, cursor đúng; replay pull ổn định |
| Proxy | API/Worker không publish host ports, Nginx restart | Auth vẫn bắt buộc; request tới API ngừng trả 502; data không mất; retry cùng UUID sau recovery không duplicate |
| Realtime/tenant | WebSocket SignalR thật qua Nginx, tenant A/B đều hoạt động | Snapshot có invoice đã commit; B không nhận event A và ngược lại; B không GET/pull invoice A |
| Missed event | Commit/publish khi client không nối hub, reconnect sau đó | Pull từ sequence cũ trả đúng snapshot/operation một lần |
| Outbox replay | Đưa Published outbox về Pending để mô phỏng publish-before-mark replay | Hai delivery giữ nguyên messageId/sequence; durable change/invoice không duplicate |
| Outbox recovery | Broker unavailable sau DB commit | Invoice/operation/change/outbox Pending còn nguyên; broker/worker recovery publish và pull đúng một lần |
| Poison/retry | Gửi malformed JSON vào queue | Không retry vô hạn; DLQ sau 5 retry, giữ message body/id; retry queue rỗng |
| Backup/restore | Dump thật, encrypted artifact, restore vào DB và vhost riêng | Fingerprint từng bảng bằng nguồn; login/sync/pull/reverse hoạt động; replay receipt/reverse không duplicate; không ghi vào DB nguồn |
| Backup fail-closed | Dump lỗi, thiếu completion footer, truncation, corruption, sai key | Lệnh lỗi hoặc exception rõ; không trả backup hợp lệ |

### 10.1. Ngưỡng và bằng chứng

- Deadline recovery/poll mỗi boundary: 60 giây trên fixture nhỏ. Đây là ngưỡng
  engineering của test, **không phải** SLA/RTO production đã được phê duyệt.
- Bootstrap container tối đa 180 giây; mỗi subprocess Docker tối đa 8 phút.
  Thời gian tải image/build không được diễn giải thành RTO khôi phục dữ liệu.
- Test dùng polling theo trạng thái thật và monotonic stopwatch cho process
  bên ngoài; không dùng clock mobile để quyết định thứ tự ghi sổ.
- Không dùng sleep dài hoặc retry cả test để che fault. Đường phục hồi được
  quan sát tới commit/Published và drain cả delivery unacked trước khi đếm.
- Tối thiểu 23 test phải chạy và pass, không skip. Runner mặc định có watchdog
  hang 20 phút; mỗi lần chạy có TRX riêng và cleanup stack riêng.
- `scripts/test-resilience.ps1 -ColdStartOnly` chỉ chạy cold-start gate riêng,
  ghi ba mẫu trong `cold-start.json` và API/Worker logs có timestamp. Pass một
  test này không thay cho full gate, đo trên VPS đích hoặc soak đủ 24 giờ.
- Manifest ghi commit/working tree và SHA-256 source trước test; source thay đổi
  trong quá trình chạy thì gate fail thay vì chứng nhận nhầm build.
- Test hạ tầng được skip trong `dotnet test` thông thường nếu chưa chạy runner;
  **skip không phải bằng chứng release**. Các test mã hóa thuần không cần Docker.
- Artifact chứa dữ liệu tổng hợp; không log token/secret hoặc dùng PII thật.

### 10.2. Giới hạn và release blocker còn lại

Đây là gate backend/infrastructure local, không phải E2E Flutter/SQLCipher.
Việc mất mạng/batch và không đọc response được mô phỏng bằng HTTP client test;
nó không chứng minh app đã giữ outbox qua OS process kill hoặc thiết bị hỏng.
Outbox replay kiểm tra message identity/durable state; dedup trên SQLite và
advance cursor sau local transaction vẫn cần test trên thiết bị.

Nginx fixture chỉ HTTP loopback, không chứng minh HTTPS, forwarded-header trust,
ẩn origin IP hay gia hạn chứng chỉ. Không chạy script chaos lên production.
Backup ngoài VM thật (upload/download, key recovery, retention/quota, alert), mất VM,
backup khi workload production đang ghi, restore đầy đủ dữ liệu production, Shorebird
patch/rollback và mobile native compatibility vẫn cần bằng chứng riêng theo
staging/release gate. Không gộp các mục chưa test thành trạng thái PASS.

## 11. Gate mobile trên development đang chạy

Từ `backend`, chạy trên Windows:

```powershell
.\scripts\test-development-mobile.ps1 -AllowDevelopmentInterruption
```

Đây là gate **opt-in có gián đoạn**, khác gate cô lập ở mục 10. Nó chỉ nhận
emulator Android local và container development thuộc workspace hiện tại;
API phải là `Development`, cổng `5099` và các cổng publish phải là loopback.
Không chạy trên production. Ngoại lệ dùng tenant demo development hiện có
chỉ áp dụng khi người vận hành đã đồng ý; không dùng dữ liệu khách production.
Runner không xóa volume hoặc sửa/xóa hóa đơn đã đồng bộ.

Trước deploy, dừng API/Worker, dump database development thực, mã hóa AES-GCM,
đọc lại và restore vào MySQL disposable có label riêng. Fingerprint **tất cả
bảng** phải bằng nguồn. Chỉ sau bước này mới rebuild API/Worker bằng Compose
hiện có với `--no-deps`; MySQL/RabbitMQ và volume hiện có không bị recreate.

Mobile chạy `integration_test/development_resilience_test.dart` với SQLCipher,
`InvoiceStore`, `RemoteInvoiceRepository`, API client và SignalR client thực:

| Boundary | Assertion |
|---|---|
| Outbox INSERT lỗi sau cập nhật invoice trong transaction | Invoice vẫn Draft, không có outbox; chỉ xóa draft do test vừa tạo |
| API thực sự ngừng, submit offline | 6 invoice/outbox Pending; submit lại bị chặn; frozen UUID/payload giữ nguyên |
| Durable Uploading rồi host force-stop PID Android | Pha resume có PID mới, mở lại SQLCipher; Uploading về Pending, UUID/payload byte-for-byte không đổi |
| Batch mất response sau acceptance thứ 3, rồi API ngừng | 2 Queued, 4 Pending trong batch test; không invoice nào được coi là Synced |
| Worker vẫn dừng khi API phục hồi | Cả 6 Queued, không được mark Synced chỉ vì HTTP 202 |
| SignalR handshake thật rồi khởi động Worker | Có DataCreated của operation test; receipt Completed làm local invoice/outbox Synced |
| Bỏ 202 thật rồi retry | Operation UUID/payload không đổi; server verifier chứng minh không ghi trùng |
| RabbitMQ ngừng trước acceptance | API 503, local Pending; broker recovery đồng bộ thành công |
| Hub đã ngắt khi hai invoice cuối commit | Pull page size 2 từ sequence cũ trả operation bị lỡ, strictly increasing, không sequence trùng |
| Đối chiếu cuối | 8 invoice test local/outbox Synced; mỗi invoice server có một detail, một Completed receipt, một server change và một Published DataCreated outbox |

Pha prepare có 3 checkpoint, resume có 8 checkpoint; driver phải pass cả hai,
không được chỉ chạy pha prepare để kết luận thành công. Recovery deadline mỗi
điều kiện là 75 giây, control request 90 giây, mỗi subprocess host 12 phút.
Đây không phải SLA production; không retry cả suite tự động.

Bằng chứng ở `backend/artifacts/development-mobile/<run>/`: TRX prepare/verify/revoke,
`prepare.json`, `resume.json`, `device-process.json`, `backup-restore.json`,
`manifest.json`, backup mã hóa và key được Windows DPAPI bảo vệ cho tài khoản
đã chạy runner. Source thay đổi trong lúc chạy làm gate fail. Token test chỉ
ở file defines tạm; session bị revoke và file bị xóa trong finally. App debug
thường được cài lại bằng `adb install -r`, không clear SQLCipher/secure storage.
Các invoice test có prefix `INV-TEST-<run>-`; lịch sử được giữ để đối chiếu.

Giới hạn: force-stop sau durable Uploading, **không** kill trong mọi instruction
của SQLite transaction hoặc giả lập mất điện/disk full/corruption/key loss.
Pull này kiểm tra API và identity sau missed event, không chứng nhận toàn bộ
mobile pull-coordinator/cursor transaction hay multi-device conflict.
Không thao tác MySQL crash/Nginx trên stack development; các boundary đó thuộc
gate cô lập mục 10. Không test giấy in, backup ngoài VM, TLS hoặc Shorebird OTA. Bản debug
thường không phải APK nền Shorebird; cần cài lại bản nền trước thử patch.

### 11.1. Bằng chứng development ngày 2026-11-01

Run `10e711ecd6964c088f22f96e0e7273c4` kết thúc lúc
`2026-11-01T09:23:14.6712001Z`, exit code 0. Prepare/resume pass đủ 3+8
checkpoint; verifier MySQL và revoke session đều pass, không skip.
8 invoice/local outbox Synced; mỗi server invoice có đúng một detail,
Completed operation, server change và Published DataCreated outbox.
Backup development restore khớp fingerprint cả 18 bảng.

Host chứng minh force-stop PID `10413` và resume bằng PID `10551`.
Manifest ghi SHA-256 của 488 source file, `sourceUnchanged: true`,
`cleanupCompleted: true`, `passed: true`. API/Worker/MySQL/RabbitMQ đều chạy,
ready 200, app debug thường mở lại. Control port forwarding, private defines
và container restore disposable đã được gỡ; forwarding API 5099 được giữ cho app.
Bằng chứng tại `backend/artifacts/development-mobile/10e711ecd6964c088f22f96e0e7273c4/`.
Sau restore đã attach app thường, hot reload thành công qua Dart SDK MCP và
`get_runtime_errors` không trả lỗi. Helper attach được dừng; app vẫn chạy.

Các lần thử trước có lỗi harness/snapshot và một lần bị source gate chặn khi
Flutter tự thêm ignore rule `.widget_preview/`; chúng không được tính là pass.
Không bỏ qua source check hoặc sửa manifest cũ để chứng nhận lại.
