# TÀI LIỆU QUY TẮC NGHIỆP VỤ & BUSINESS LOGIC
**Mục tiêu:** Định nghĩa rõ ràng các luồng hoạt động kinh doanh, phân quyền và chuẩn mực kế toán. AI Agent bắt buộc phải code theo các rào cản logic này.

---

## 1. PHÂN QUYỀN (ROLE-BASED ACCESS CONTROL - RBAC)
Hệ thống sử dụng JWT Authentication với 3 vai trò (Roles) phân lập:

*   **`SuperAdmin` (Quản trị nền tảng SaaS):**
    *   Xem, tạo, sửa, vô hiệu hóa và xóa tài khoản `TenantAdmin`; tài khoản phải được gán đúng `TenantId`.
    *   Xem, tạo, sửa, vô hiệu hóa và xóa các cửa hàng (Tenants).
    *   Xóa tài khoản hoặc tenant có dữ liệu nghiệp vụ được hiểu là xóa mềm; không xóa vật lý lịch sử/audit và phải thu hồi các phiên đăng nhập liên quan.
    *   Cấu hình `Global Feature Flags` (bật/tắt module Kho, Chấm công cho toàn hệ thống).
*   **`TenantAdmin` (Chủ cửa hàng):**
    *   Được sử dụng các chức năng của `Staff` và có toàn quyền CRUD trên dữ liệu thuộc `TenantId` của mình, theo các giới hạn bên dưới.
    *   Xem, tạo, sửa, vô hiệu hóa và xóa tài khoản Staff. Nếu tài khoản đã được tham chiếu trong lịch sử nghiệp vụ/audit, thao tác xóa phải là xóa mềm.
    *   Xem, thêm, sửa và vô hiệu hóa sản phẩm; chỉ được xóa sản phẩm chưa có dữ liệu lịch sử.
    *   Giá sản phẩm và `UnitPrice` phải lớn hơn 0. Tăng/giảm giá hàng loạt dùng độ lớn dương cùng hướng thao tác riêng (`Increase`/`Decrease`); UI quyết định cách hiển thị dấu.
    *   Nghiệp vụ tính thuế chưa thuộc v1; giữ `TaxRate` và `TaxAmount` bằng 0 cho đến khi triển khai phần mở rộng thuế.
    *   Xem, tạo, sửa và xóa khách hàng. Chỉ xóa vật lý khách hàng chưa được tham chiếu bởi hóa đơn; nếu đã có lịch sử, phải vô hiệu hóa hoặc xóa mềm để giữ nguyên tham chiếu.
    *   Cấu hình thông tin hiển thị cửa hàng, mẫu hóa đơn và cách hiển thị sản phẩm.
    *   Đổi giá hàng loạt (Bulk Update).
    *   Duyệt thực hiện bút toán đảo ngược (hủy/sửa hóa đơn đã đồng bộ).
    *   Cấu hình `Tenant Feature Flags` (tự động làm tròn tiền, in Bluetooth).
*   **`Staff` (Nhân viên/Thu ngân):**
    *   Xem/tìm kiếm sản phẩm và xem giá; không được sửa giá gốc sản phẩm.
    *   Xem/tìm kiếm và tạo khách hàng để phục vụ lập hóa đơn; không được sửa hoặc xóa khách hàng.
    *   Lập hóa đơn mới.
    *   Được in hóa đơn trên thiết bị client; Bluetooth phụ thuộc cấu hình cửa hàng/feature flag.
    *   Không được quản lý tài khoản, cấu hình cửa hàng/feature flag hoặc thay đổi dữ liệu sản phẩm.

---

## 2. NGHIỆP VỤ KẾ TOÁN: BÚT TOÁN ĐẢO NGƯỢC (REVERSE ENTRY)
**Quy tắc bất khả xâm phạm:** KHÔNG dùng lệnh `UPDATE` hay `DELETE` trên bảng `Invoices` và `InvoiceDetails` sau khi hóa đơn đã được đồng bộ.

*   **Luồng Hủy Hóa Đơn:**
    1. User (có quyền) chọn hóa đơn `INV-001` (Trị giá: 1,000,000đ).
    2. API tạo một hóa đơn mới `INV-002` (Trị giá: -1,000,000đ, số lượng items ÂM).
    3. Cập nhật trạng thái của cả hai hóa đơn thành `Reversed` (Đã hủy/đối trừ).
    4. Cột `OriginalInvoiceId` của `INV-002` trỏ về `INV-001`.
*   **Luồng Sửa Hóa Đơn:**
    1. Gọi "Luồng Hủy Hóa Đơn" cho hóa đơn cũ.
    2. Tạo một hóa đơn `INV-003` hoàn toàn mới với các thông tin đã được chỉnh sửa.

---

## 3. ĐỒNG BỘ PHÂN TÁN (OFFLINE-FIRST & SYNC LOGIC)
*   **Trạng thái máy tính tiền (App Flutter):** Luôn lấy SQLite làm nguồn dữ liệu (Source of Truth) cho giao diện để đảm bảo tốc độ 0ms và chống rớt mạng.
*   **Outbox Pattern:**
    1. Thu ngân chốt hóa đơn. App lưu hóa đơn vào SQLite với cờ `SyncStatus = Pending`.
    2. App Background Worker ping đến server. Nếu có mạng, bốc toàn bộ list `Pending` đẩy qua API `/api/invoices/sync`.
*   **Realtime Broadcast:**
    1. Server nhận data, insert MySQL thành công.
    2. Server gọi SignalR Hub bắn event `OnDataUpdated` xuống TẤT CẢ các thiết bị khác thuộc **cùng Tenant Group**.
    3. Các thiết bị này nhận event, tự động gọi API pull dữ liệu mới về gộp vào SQLite.

---

## 4. QUY TẮC ĐA NGÔN NGỮ (I18N & L10N)
*   **API (Backend):** Các file Exception hoặc Response Message tuyệt đối không hardcode text (Ví dụ: `throw new Exception("Sản phẩm không tồn tại")`). BẮT BUỘC sử dụng Error Code chuẩn (Ví dụ: `ERR_PRODUCT_NOT_FOUND`).
*   **Mobile (Frontend):** Bắt các Error Code này và dùng `easy_localization` map với file JSON (`vi.json`, `en.json`) để render ra ngôn ngữ cấu hình trên máy người dùng.

---

## 5. CỜ TÍNH NĂNG (FEATURE FLAGS LOGIC)
Dữ liệu cấu hình lấy từ bảng `AppSettings`.
*   App sẽ khóa hoặc ẩn các tab trên UI dựa vào cấu hình.
*   Ví dụ: Nếu `AppSettings` trả về `{"RequireCustomerPhone": true}`, nút "Thanh toán" sẽ bị disable nếu field Số điện thoại bị bỏ trống.

---

## 6. QUY TẮC MOBILE

### 6.1. Trải nghiệm và khả năng tiếp cận
*   Mobile hướng đến người dùng ít kinh nghiệm công nghệ, người không quen nhập liệu trên điện thoại và người trung niên/cao tuổi.
*   Luồng bán hàng, tạo sản phẩm và nhập khách hàng phải ngắn, dễ hiểu, ưu tiên chọn/tìm/quét và giá trị mặc định hợp lý thay cho nhập nhiều trường.
*   Chữ phải rõ, dễ đọc; vùng chạm đủ lớn; nút tăng/giảm số lượng `-`/`+` phải dễ nhìn và dễ thao tác. Không dùng màu sắc làm tín hiệu duy nhất; trạng thái quan trọng cần có nhãn/icon hoặc mô tả trợ năng.
*   Tìm kiếm, bộ lọc và sắp xếp phải dễ tìm thấy. Danh sách lớn tải lười theo cursor keyset, nạp trước tối đa một trang tiếp theo để cuộn liên tục, không tải toàn bộ danh sách một lần.
*   UI phải co giãn theo kích thước màn hình và độ dài bản dịch; không cắt, tràn hoặc chồng lấn nội dung khi đổi ngôn ngữ. Mọi text hiển thị và lỗi dùng `easy_localization`, không hardcode trong widget.

### 6.2. Offline, lưu local và đồng bộ
*   SQLite/SQLCipher là nguồn dữ liệu cho UI mobile. Tạo hoặc chỉnh sửa sản phẩm, khách hàng và hóa đơn phải được lưu local trước để người dùng tiếp tục thao tác khi mất mạng.
*   Mỗi thay đổi cần đồng bộ được ghi cùng local outbox trong transaction, có `ClientOperationId` ổn định và idempotency; retry phải dùng lại operation ID để không tạo bản ghi trùng.
*   Khi có mạng, app tự gửi outbox nền lên API/RabbitMQ và hỗ trợ đồng bộ thủ công. Đồng bộ không được khóa luồng bán hàng hoặc làm mất thay đổi người dùng đang nhập.
*   Phân biệt rõ trạng thái đã lưu trên thiết bị, đang gửi, đã được server nhận vào queue, đã đồng bộ hoàn tất và lỗi cần xử lý. `Queued` không đồng nghĩa với `Synced`; xác nhận hoàn tất theo event/pull của sync contract.
*   UI thể hiện riêng trạng thái có mạng, mất mạng, đang đồng bộ lên, đang nhận dữ liệu xuống, chờ xử lý, hoàn tất và lỗi đồng bộ bằng semantic color token riêng cho từng trạng thái, kèm nhãn/icon. Token lấy từ theme, không khai báo màu rải rác; không dùng màu làm tín hiệu duy nhất. Không báo đã đồng bộ thành công trước khi có xác nhận server.
*   Mất mạng hoặc lỗi tạm thời giữ nguyên local data/outbox và tự retry theo sync policy. Lỗi nghiệp vụ bị từ chối phải hiển thị lỗi đã localization, giữ dữ liệu để người dùng sửa hoặc xử lý; không retry vô hạn.
*   Product, customer và invoice được gửi theo command contract hiện có để backend xử lý qua RabbitMQ; client không giả định `202 Queued` là dữ liệu đã commit. SignalR báo thay đổi cho thiết bị cùng tenant; client pull/upsert theo `ServerSequence`, deduplicate và không coi event là nguồn sự thật duy nhất.

### 6.3. Cache, tìm kiếm và danh sách
*   Cache local của hóa đơn đã đồng bộ được giữ tối đa 30 ngày; cache customer được giữ tối đa 3 tháng tính từ lần đồng bộ thành công gần nhất. Khi dữ liệu đã hết hạn cần lấy bản mới từ server khi có mạng, đồng thời hiển thị dữ liệu local phù hợp trong lúc chờ nếu có.
*   Thời hạn trên chỉ áp dụng cho cache local đã đồng bộ. Không xóa bản nháp, dữ liệu đang có thay đổi chưa đồng bộ hoặc outbox. Chỉ hóa đơn/nháp đã gắn với customer được người dùng lưu/chọn mới được giữ snapshot customer tối thiểu để hiển thị; không giữ customer cache đầy đủ quá hạn. Khách vãng lai (`CustomerId = null`) luôn để trống thông tin customer trên hóa đơn/bản in và không có snapshot customer. Không áp dụng thời hạn local lên dữ liệu server, audit hay nghĩa vụ lưu trữ kế toán.
*   Tìm sản phẩm/khách hàng/hóa đơn phải trả kết quả local ngay; khi online, đồng thời truy vấn server theo cùng từ khóa/bộ lọc. Server là nguồn xác nhận cho dữ liệu đã đồng bộ; dữ liệu local chưa đồng bộ vẫn được giữ và hiển thị riêng, không bị response server ghi đè.
*   Chuẩn hóa truy vấn giống nhau ở local/server: trim khoảng trắng, Unicode NFC, không phân biệt hoa thường; chuỗi rỗng sau chuẩn hóa tương đương không có filter. Text fields dùng substring match; barcode/UUID dùng exact match theo API contract. Tìm sản phẩm theo tên/SKU/barcode; khách hàng theo tên/mã/số điện thoại; hóa đơn theo số hóa đơn hoặc barcode/ID hóa đơn. Local search phải dùng cùng field và semantics với API.
*   Khi server trả kết quả, gộp theo entity ID, loại trùng và chỉ thay bản local đã đồng bộ bằng version server mới hơn hoặc bằng; không ghi đè entity có local mutation/outbox pending. Kết quả offline giữ nguyên khi request server lỗi và UI phải báo rõ dữ liệu có thể chưa mới.
*   Thay từ khóa, bộ lọc, sort hoặc tenant phải hủy/đánh dấu stale request cũ, xóa cursor cũ và bắt đầu truy vấn mới. Response của truy vấn cũ không được cập nhật list hiện hành.
*   Danh sách lớn dùng cursor opaque và page size giới hạn; sort phải deterministic với khóa phụ duy nhất. Nạp trước đúng một trang tiếp theo khi gần cuối list, không gọi lặp cùng cursor; chỉ commit cursor sau khi page được xử lý thành công. Cursor server không dùng thay `ServerSequence` của sync pull.
*   Realtime/pull upsert theo ID/version, deduplicate và không làm nhảy vị trí cuộn. Nếu thay đổi ảnh hưởng bộ lọc hoặc thứ tự hiện tại, cập nhật list có kiểm soát và làm mới cursor khi cần.

### 6.4. Nhập liệu và lập hóa đơn
*   Hóa đơn đang nhập được tự lưu nháp local sau mỗi thay đổi có thể lưu; khi app bị đóng, gián đoạn hoặc có cuộc gọi, người dùng có thể tiếp tục từ danh sách/tìm kiếm nháp. Nháp chưa chốt không được gửi như hóa đơn hoàn tất.
*   Khi lập hóa đơn, mặc định là khách vãng lai: hóa đơn có `CustomerId = null` và phần thông tin khách hàng trên hóa đơn/invoice print để trống.
*   Có nút dễ thấy `Chọn khách hàng đã lưu` để mở danh sách khách hàng, tìm kiếm và chọn nhanh; kết quả local hiển thị trước, tìm server khi có mạng theo quy tắc offline/search hiện hành. Chỉ customer được chọn từ danh bạ mới được gắn vào hóa đơn.
*   Tạo customer dùng luồng riêng với nút `Lưu khách hàng`/`Lưu vào danh bạ`; chỉ thao tác xác nhận lưu này mới tạo customer master record. Không có form nhập thông tin khách tạm trên hóa đơn và không tự tạo customer khi lưu nháp hoặc chốt hóa đơn. Sau khi lưu thành công, customer được chọn cho hóa đơn hiện tại.
*   Nếu người dùng hủy/đóng luồng tạo khách hoặc không chọn customer đã lưu, hóa đơn tiếp tục là khách vãng lai; không giữ thông tin nhập dở trong invoice payload, bản in hoặc lịch sử hóa đơn. Lưu nháp hóa đơn không đồng nghĩa với lưu khách hàng.
*   Tạo customer vào danh bạ tuân theo RBAC: Staff được tạo nhưng không sửa/xóa; TenantAdmin có quyền quản lý theo giới hạn tại mục 1. Nếu tạo khi offline, lưu local và outbox trước, retry idempotent như các dữ liệu khác.
*   Hóa đơn bán thông thường không chấp nhận số lượng nhỏ hơn hoặc bằng 0; cảnh báo trực quan và chặn chốt hóa đơn. Hóa đơn đảo là luồng hệ thống riêng, được phép lưu số lượng/giá trị âm theo reverse-entry rule.
*   Giá nhập không được âm. Quy tắc nghiệp vụ hiện hành vẫn yêu cầu giá sản phẩm và `UnitPrice` lớn hơn 0; bulk increase/decrease dùng độ lớn dương và giá cuối cùng phải lớn hơn 0.
*   Trường giá và số điện thoại phải mở bàn phím số phù hợp. Giá hiển thị theo định dạng VND của thiết bị; dữ liệu gửi API phải là giá trị số theo contract, không gửi chuỗi đã định dạng theo locale.
*   Nút tăng/giảm số lượng phải thao tác nhanh, cập nhật tổng tiền tức thời và giữ validation nhất quán. Cho phép tăng/giảm giá một hoặc nhiều sản phẩm theo quyền RBAC và quy tắc bulk update hiện hành.
*   Trên hóa đơn tạo mã barcode ổn định từ định danh hóa đơn để tìm/tra cứu nhanh; không mã hóa PII. Quét barcode phải tìm local trước và gọi server khi cần, theo cùng quy tắc quyền/tenant.

### 6.5. Ngôn ngữ, ngày giờ và tiền tệ
*   Mobile hỗ trợ các ngôn ngữ được cấu hình bằng `easy_localization`; đổi ngôn ngữ không được làm vỡ layout, mất nội dung hoặc khiến thao tác chính không dùng được.
*   Ngày giờ hiển thị theo locale và múi giờ hiện hành của thiết bị. Timestamp gửi/nhận qua API tuân theo contract backend: UTC+0, ISO 8601 có ký hiệu UTC (`Z`). Không dùng đồng hồ client để quyết định thứ tự đồng bộ hoặc ghi sổ.
*   Số tiền hiển thị theo VND và locale thiết bị; phép tính/lưu trữ dùng kiểu decimal theo contract, không dùng floating point.

### 6.6. Lịch sử, in hóa đơn và cập nhật ứng dụng
*   Người dùng chỉ xem được lịch sử thay đổi phù hợp quyền RBAC và dữ liệu audit server cung cấp. Hóa đơn đã đồng bộ vẫn bất biến; sửa/hủy phải dùng reverse/correction, không ghi đè lịch sử.
*   TenantAdmin có thể cấu hình mẫu/thông tin hiển thị hóa đơn để in. Kết nối máy in Bluetooth tuân theo feature flag/cấu hình tenant; lỗi máy in không được rollback hóa đơn hoặc thanh toán.
*   App kiểm tra chính sách phiên bản/cập nhật từ xa hiện có, thông báo rõ phiên bản mới và hướng dẫn cập nhật. Chỉ bắt buộc cập nhật khi cấu hình phiên bản tối thiểu hoặc yêu cầu bảo mật quy định; OTA patch phải tương thích với native version và API contract.