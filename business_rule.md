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
    *   Xem, thêm, sửa và vô hiệu hóa sản phẩm gốc/biến thể; chỉ được xóa sản phẩm hoặc biến thể chưa có dữ liệu lịch sử.
    *   Danh mục sản phẩm thuộc `TenantId` hiện tại. Chủ cửa hàng quản lý danh mục trong Cài đặt; chỉ xóa cứng danh mục chưa có bất kỳ sản phẩm nào (kể cả ngừng bán) và không còn danh mục con. Danh mục có sản phẩm chỉ bật/tắt; tắt giữ nguyên dữ liệu và ẩn tab khỏi màn hình sản phẩm. Mobile không bắt nhập mã danh mục: để trống tự sinh `DM-<UUID viết hoa, bỏ gạch nối>`; mã tự sinh được giữ khi lỗi để thử lại, mã nhập tay được giữ nguyên.
    *   Mobile phải có ít nhất một danh mục đang hoạt động trước khi thêm sản phẩm. Nếu chưa có, mở quản lý danh mục cho TenantAdmin; Staff được hướng dẫn liên hệ chủ cửa hàng. Danh sách có các tab danh mục đang hoạt động dưới ô tìm kiếm, không có tab Tất cả. Hiển thị/tìm kiếm chỉ trong danh mục đang chọn; sản phẩm mới tự nhận `categoryId` của tab đó.
    *   Tạo sản phẩm gốc được nhập đặc điểm như độ dày, thương hiệu ngay trong lần lưu đầu tiên; lưu `extraData` lên product và default variant, không bắt người dùng lưu rồi sửa lại.
    *   Mỗi sản phẩm gốc có một hoặc nhiều biến thể bán được. SKU, barcode, đơn vị bán, giá niêm yết, số lượng/khối lượng quy đổi và giá vốn thuộc về biến thể; SKU duy nhất trong tenant và các giá trị dùng tính tiền phải lớn hơn 0.
    *   Áp dụng cho mọi sản phẩm, gồm sắt thép và phụ kiện: `giá bán = giá niêm yết × số lượng/khối lượng quy đổi`, làm tròn tối đa 4 chữ số thập phân. `Unit` là đơn vị bán (ví dụ cây, cái, cặp); hệ số quy đổi là lượng tính giá cho một đơn vị bán. Sản phẩm cũ mặc định hệ số 1 và giá niêm yết bằng giá bán hiện tại.
    *   Ngoại lệ chủ cửa hàng phê duyệt ngày 2026-11-04: danh mục bật `RoundSalePriceToThousands` lưu giá bán bằng `max(1000, floor((giá niêm yết × hệ số + 500) / 1000) × 1000)` VND. Tính trực tiếp trên tích chính xác, không làm tròn trung gian; 19.400 → 19.000, 19.500 → 20.000. Chỉ bật cho danh mục Sắt thép được chọn, không suy luận từ tên/brand; mặc định mọi danh mục khác tắt. Áp dụng tạo/sửa/đổi giá cả online/offline, giữ nguyên giá niêm yết/hệ số và snapshot hóa đơn cũ. Xem [ADR-015](architecture/adr/ADR-015-category-sale-price-rounding.md).
    *   Mã hàng tự sinh mới là `MH-` + 12 ký tự, số hóa đơn mới là `INV-` + 12 ký tự theo [ADR-014](architecture/adr/ADR-014-short-business-codes.md). UUID đầy đủ vẫn là identity; không đổi mã nhập tay, mã đã lưu hoặc số hóa đơn lịch sử. Mã danh mục không đổi.
    *   Tăng/giảm giá hàng loạt điều chỉnh giá niêm yết, sau đó tự tính lại giá bán theo hệ số của từng product/variant; cho phép chọn giá gốc/product, từng variant hoặc kết hợp cả hai trong cùng giao dịch. Mỗi target được tính riêng; giá product và default variant là cùng giá nên nếu cùng chọn chỉ điều chỉnh một lần. `BasePrice`/giá bán lưu trên server luôn là giá sau tính toán.
    *   Màn hình catalog và bán hàng hiển thị giá bán đã tính, không ghép thêm “/ đơn vị tính”; đơn vị vẫn được dùng riêng cho thông tin sản phẩm và số lượng hóa đơn.
    *   Nghiệp vụ tính thuế chưa thuộc v1; giữ `TaxRate` và `TaxAmount` bằng 0 cho đến khi triển khai phần mở rộng thuế.
    *   Xem, tạo, sửa và xóa khách hàng. Chỉ xóa vật lý khách hàng chưa được tham chiếu bởi hóa đơn; nếu đã có lịch sử, phải vô hiệu hóa hoặc xóa mềm để giữ nguyên tham chiếu.
    *   Cấu hình thông tin hiển thị cửa hàng, mẫu hóa đơn và cách hiển thị sản phẩm.
    *   Đổi giá hàng loạt (Bulk Update).
    *   Duyệt thực hiện bút toán đảo ngược (hủy/sửa hóa đơn đã đồng bộ).
    *   Cấu hình `Tenant Feature Flags` (tự động làm tròn tiền, in Wi-Fi, tùy chọn Bluetooth chưa có transport).
*   **`Staff` (Nhân viên/Thu ngân):**
    *   Xem/tìm kiếm sản phẩm, chọn biến thể bán được và xem giá của biến thể; không được sửa giá.
    *   Xem/tìm kiếm và tạo khách hàng để phục vụ lập hóa đơn; không được sửa hoặc xóa khách hàng.
    *   Lập hóa đơn mới.
    *   Được in hóa đơn có quyền đọc qua PC Wi-Fi khi `EnableWifiPrint` đã được server xác nhận; không sửa quyền in/cấu hình ngân hàng. Bluetooth chưa triển khai.
    *   Không được quản lý tài khoản, cấu hình cửa hàng/feature flag hoặc thay đổi dữ liệu sản phẩm.

---

## 2. NGHIỆP VỤ KẾ TOÁN: BÚT TOÁN ĐẢO NGƯỢC (REVERSE ENTRY)
**Quy tắc bất khả xâm phạm:** KHÔNG dùng lệnh `UPDATE` hay `DELETE` trên bảng `Invoices` và `InvoiceDetails` sau khi hóa đơn đã được đồng bộ.

*   **Luồng Hủy Hóa Đơn:**
    1. User (có quyền) chọn hóa đơn `INV-001` (Trị giá: 1,000,000đ).
    2. API tạo một hóa đơn mới `INV-002` (Trị giá: -1,000,000đ, số lượng items ÂM).
    3. Cập nhật trạng thái của cả hai hóa đơn thành `Reversed` (Đã hủy/đối trừ).
    4. Cột `OriginalInvoiceId` của `INV-002` trỏ về `INV-001`.
    5. `InvoiceStatus` chỉ biểu diễn trạng thái nghiệp vụ: `Active` hoặc `Reversed`. Khi đảo, hóa đơn gốc và bút toán Reverse cùng chuyển thành `Reversed`; bút toán Correction mới là `Active`.
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
*   Thuộc tính bổ sung riêng của sản phẩm/cửa hàng như chất liệu, kích thước hoặc độ dày không phải feature flag; lưu theo contract `ExtraDataJson` trong [database_design.md](database/database_design.md). Cấu hình làm thay đổi hành vi ứng dụng vẫn phải lưu trong `AppSettings`.

---

## 6. QUY TẮC MOBILE

### 6.1. Trải nghiệm và khả năng tiếp cận
*   Mobile hướng đến người dùng ít kinh nghiệm công nghệ, người không quen nhập liệu trên điện thoại và người trung niên/cao tuổi.
*   Luồng bán hàng, tạo sản phẩm và nhập khách hàng phải ngắn, dễ hiểu, ưu tiên chọn/tìm/quét và giá trị mặc định hợp lý thay cho nhập nhiều trường.
*   Chữ phải rõ, dễ đọc; vùng chạm đủ lớn; nút tăng/giảm số lượng `-`/`+` phải dễ nhìn và dễ thao tác. Không dùng màu sắc làm tín hiệu duy nhất; trạng thái quan trọng cần có nhãn/icon hoặc mô tả trợ năng.
*   Tìm kiếm, bộ lọc và sắp xếp phải dễ tìm thấy. Danh sách lớn tải lười theo cursor keyset, nạp trước tối đa một trang tiếp theo để cuộn liên tục, không tải toàn bộ danh sách một lần.
*   UI phải co giãn theo kích thước màn hình và độ dài bản dịch; không cắt, tràn hoặc chồng lấn nội dung khi đổi ngôn ngữ. Mọi text hiển thị và lỗi dùng `easy_localization`, không hardcode trong widget.

### 6.2. Offline, lưu local và đồng bộ
*   Ngừng bán sản phẩm gốc ngừng tất cả biến thể; bán lại sản phẩm gốc bật tất cả biến thể, bao gồm biến thể đã ngừng riêng trước đó. Ngừng/bán lại một biến thể (kể cả default hoặc biến thể active cuối cùng) không đổi gốc hoặc biến thể khác. Đây là quy tắc lifecycle thay cho hạn chế ngừng biến thể active cuối cùng trước đây; hạn chế xóa cứng và lịch sử hóa đơn vẫn giữ nguyên. Khả năng bán thực tế yêu cầu cả gốc và biến thể active.
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
*   Danh sách hóa đơn hiển thị số hóa đơn, tổng tiền kèm tiền tệ và ngày giờ tạo. Người dùng nhập số để tìm hoặc quét barcode chứa đúng số hóa đơn để mở nhanh chi tiết; không tự mở hóa đơn chỉ khớp một phần. Tra cứu offline chỉ bao gồm dữ liệu đã lưu trên thiết bị.
*   Số hóa đơn được tự sinh duy nhất ngay khi tạo nháp, hỗ trợ nhiều thiết bị offline và giữ nguyên qua lưu nháp/chốt/retry. Không bắt người dùng nhập số hóa đơn.
*   Chi tiết hóa đơn và mẫu in vẫn có ba dòng Tên khách hàng, Địa chỉ, Số điện thoại. Khi `CustomerId = null`, giữ các dòng này trống và đủ chỗ ghi tay; không thay bằng tên khách vãng lai, thông tin cửa hàng hoặc dữ liệu customer trước đó.
*   Hóa đơn đang nhập được tự lưu nháp local sau mỗi thay đổi có thể lưu; khi app bị đóng, gián đoạn hoặc có cuộc gọi, người dùng có thể tiếp tục từ danh sách/tìm kiếm nháp. Nháp chưa chốt không được gửi như hóa đơn hoàn tất.
*   Mở composer trống không tạo nháp. Chỉ lưu khi đã chọn hàng hoặc có phí dịch vụ; bỏ hết hàng/phí thì loại nháp rỗng. Người dùng được xóa nháp local sau xác nhận; tuyệt đối không dùng thao tác này cho hóa đơn đã chốt, đang đồng bộ hoặc đã sync.
*   Picker chỉ hiển thị variant bán được như danh sách sản phẩm; hiện tên/đặc điểm extraData để phân biệt variant có cùng tên gốc. Giỏ hàng, chi tiết và bản in giữ snapshot. Quantity được nhập trực tiếp hoặc +/−, phải >0 và tối đa 4 số thập phân; nút bỏ hàng là thao tác riêng.
*   Service blocks cấu hình trong mẫu là danh mục phí: nhập tiền trên từng invoice, mặc định 0, không âm, giữ snapshot ID/nhãn/số tiền. `serviceTotal` cộng vào total, không giả phí thành product/rounding/discount. Reverse đảo dấu mọi khoản phí, correction giữ lịch sử cũ và lưu bộ phí mới trong cùng transaction. Thay/xóa mẫu không xóa phí hóa đơn cũ.
*   Khi lập hóa đơn, mặc định là khách vãng lai: hóa đơn có `CustomerId = null` và phần thông tin khách hàng trên hóa đơn/invoice print để trống.
*   Có nút dễ thấy `Chọn khách hàng đã lưu` để mở danh sách khách hàng, tìm kiếm và chọn nhanh; kết quả local hiển thị trước, tìm server khi có mạng theo quy tắc offline/search hiện hành. Chỉ customer được chọn từ danh bạ mới được gắn vào hóa đơn.
*   Tạo customer dùng luồng riêng với nút `Lưu khách hàng`/`Lưu vào danh bạ`; chỉ thao tác xác nhận lưu này mới tạo customer master record. Không có form nhập thông tin khách tạm trên hóa đơn và không tự tạo customer khi lưu nháp hoặc chốt hóa đơn. Sau khi lưu thành công, customer được chọn cho hóa đơn hiện tại.
*   Nếu người dùng hủy/đóng luồng tạo khách hoặc không chọn customer đã lưu, hóa đơn tiếp tục là khách vãng lai; không giữ thông tin nhập dở trong invoice payload, bản in hoặc lịch sử hóa đơn. Lưu nháp hóa đơn không đồng nghĩa với lưu khách hàng.
*   Tạo customer vào danh bạ tuân theo RBAC: Staff được tạo nhưng không sửa/xóa; TenantAdmin có quyền quản lý theo giới hạn tại mục 1. Nếu tạo khi offline, lưu local và outbox trước, retry idempotent như các dữ liệu khác.
*   Hóa đơn bán thông thường không chấp nhận số lượng nhỏ hơn hoặc bằng 0; cảnh báo trực quan và chặn chốt hóa đơn. Hóa đơn đảo là luồng hệ thống riêng, được phép lưu số lượng/giá trị âm theo reverse-entry rule.
*   Giá nhập không được âm. Quy tắc nghiệp vụ hiện hành vẫn yêu cầu giá sản phẩm và `UnitPrice` lớn hơn 0; bulk increase/decrease dùng độ lớn dương và giá cuối cùng phải lớn hơn 0.
*   Trường giá và số điện thoại phải mở bàn phím số phù hợp. Giá hiển thị theo định dạng VND của thiết bị; dữ liệu gửi API phải là giá trị số theo contract, không gửi chuỗi đã định dạng theo locale.
*   Nút tăng/giảm số lượng phải thao tác nhanh, cập nhật tổng tiền tức thời và giữ validation nhất quán. Cho phép tăng/giảm giá một hoặc nhiều sản phẩm theo quyền RBAC và quy tắc bulk update hiện hành.
*   Trên hóa đơn tạo QR tra cứu chứa nguyên số hóa đơn, không mã hóa PII/URL. Quét QR mới hoặc barcode hóa đơn cũ phải tìm local trước và gọi server khi cần, theo cùng quy tắc quyền/tenant; bỏ qua QR ngân hàng không có tiền tố `INV-`.

### 6.5. Ngôn ngữ, ngày giờ và tiền tệ
*   Mobile hỗ trợ các ngôn ngữ được cấu hình bằng `easy_localization`; đổi ngôn ngữ không được làm vỡ layout, mất nội dung hoặc khiến thao tác chính không dùng được.
*   Ngày giờ hiển thị theo locale và múi giờ hiện hành của thiết bị. Timestamp gửi/nhận qua API tuân theo contract backend: UTC+0, ISO 8601 có ký hiệu UTC (`Z`). Không dùng đồng hồ client để quyết định thứ tự đồng bộ hoặc ghi sổ.
*   Số tiền hiển thị theo VND và locale thiết bị; phép tính/lưu trữ dùng kiểu decimal theo contract, không dùng floating point.

### 6.6. Lịch sử, in hóa đơn và cập nhật ứng dụng
*   Người dùng chỉ xem được lịch sử thay đổi phù hợp quyền RBAC và dữ liệu audit server cung cấp. Hóa đơn đã đồng bộ vẫn bất biến; sửa/hủy phải dùng reverse/correction, không ghi đè lịch sử.
*   TenantAdmin có thể cấu hình mẫu/thông tin hiển thị hóa đơn để in. Kết nối máy in Bluetooth tuân theo feature flag/cấu hình tenant; lỗi máy in không được rollback hóa đơn hoặc thanh toán.
*   App kiểm tra chính sách phiên bản/cập nhật từ xa hiện có, thông báo rõ phiên bản mới và hướng dẫn cập nhật. Chỉ bắt buộc cập nhật khi cấu hình phiên bản tối thiểu hoặc yêu cầu bảo mật quy định; OTA patch phải tương thích với native version và API contract.

### 6.7. In Wi-Fi qua PC và QR chuyển khoản
*   Phạm vi đã chốt (mở rộng 2026-11-02): Windows 10/11, Canon LBP 2900 nối cáp PC, giấy A4 hoặc A5. Điện thoại render local và gửi PNG cùng khổ giấy đã chọn qua HTTPS pin chứng chỉ trong LAN; PC dùng Windows driver theo đúng khổ. Không chuyển job qua backend/Docker, không có Bluetooth transport.
*   TenantAdmin bật/tắt `EnableWifiPrint` cho tenant, mặc định false. Staff và TenantAdmin có quyền đọc hóa đơn được in khi cờ bật và không còn draft quyền chưa xác nhận. Không in nháp chưa chốt; lỗi in không thay đổi invoice.
*   Bản in có số hóa đơn tự sinh, QR tra cứu chứa nguyên số đó (thay Code128 trên bản in mới), ngày tạo local, ba dòng khách hàng trống khi CustomerId null và chỗ ký người mua/người bán. QR tra cứu có nhãn riêng, khung 90 x 90 pt với viền trắng 8 pt mỗi cạnh; không nhầm với QR chuyển khoản. Header cho phép hai cột chữ tùy chỉnh; không thay đổi dữ liệu hóa đơn.
*   Hàng hóa trên A4 dùng bảng Tên sản phẩm, Độ dày, Số lượng, Đơn giá, Thành tiền. Độ dày lấy từ thuộc tính `thickness`, `Độ dày`, `Do day` hoặc `product.thickness` của snapshot biến thể (legacy object hoặc envelope v1; chuẩn hóa Unicode, khoảng trắng đầu/cuối và hoa/thường), tách khỏi tên, không suy đoán từ tên. Kích thước, thương hiệu và thuộc tính khác giữ trong tên hàng; phụ kiện/biến thể không có độ dày để trống cột đó. Số lượng không kèm đơn vị trên A4. Phí dịch vụ nằm ngay sau hàng hóa trong cùng bảng, độ dày trống, số lượng 1, đơn giá/thành tiền bằng phí snapshot; quy tắc ẩn phí 0 giữ nguyên, không cộng phí lần hai. Tổng giữ phía dưới theo cấu hình. Chữ A4 13 pt, dòng đầu header trái/tên cửa hàng 24 pt đậm; header phải và thông tin khác không phóng thành tiêu đề.
*   QR tra cứu bên phải số/ngày hóa đơn. QR chuyển khoản và tài khoản nhận nằm gọn bên phải header; khi chưa có tài khoản hoặc toàn bộ thông tin ngân hàng trống thì bỏ riêng QR/khối ngân hàng, vẫn có QR tra cứu. Thông tin ngân hàng nhập dở/sai vẫn phải báo lỗi.
*   Bổ sung cột STT bên trái năm cột dữ liệu A4: đánh số liên tục từ 1 cho hàng hóa rồi dịch vụ hiển thị, không reset khi sang trang hoặc tạo số cho phần tiếp nối của cùng dòng quá dài. Tiêu đề bảng lặp khi sang trang. Header thông tin bên phải căn giữa trong cột, không đổi căn phải của số tiền. Tổng chỉ xuất hiện một lần tại khối Total sau bảng theo thứ tự template, không lặp mỗi trang; chữ ký/footer sau tổng có thể nằm trang tiếp theo.
*   StoreProfile có tài khoản nhận tùy chọn: BIN 6 chữ số, tài khoản 6–34 chữ số, tên chủ tài khoản tối đa 100 ký tự. QR VietQR được tạo local từ tài khoản và tổng VND nguyên dương (tối đa 12 chữ số); không QR nhận tiền cho Reverse.
*   Theo lựa chọn của chủ cửa hàng, không truyền nội dung chuyển khoản hoặc tên người chuyển trong QR; ứng dụng ngân hàng tự quyết định nội dung mặc định. Không suy diễn rằng QR đã thanh toán: chưa triển khai thu tiền, đối soát hay trạng thái đã/chưa trả.
*   Print job độc lập invoice; retry cùng tài liệu dùng cùng UUID, không tự in lại khi trạng thái uncertain. Submitted chỉ xác nhận Windows spooler nhận job, không xác nhận giấy. In thêm bản mới phải xác nhận và kiểm tra queue khi kết quả chưa chắc chắn.
*   In thực tế hỗ trợ A4/A5; mobile chỉ hiện hai lựa chọn này, ẩn kích thước/lề và nhập ngân hàng, giữ dữ liệu cũ và QR chuyển khoản đã có. Bản gửi in có thêm Ngày in ngay dưới Ngày tạo; preview (invoice/designer) không có dòng này, không ghi ngày in vào invoice. Ngày in là thời gian chuẩn bị gửi lệnh trên điện thoại, không phải xác nhận lúc giấy ra; lưu bền theo tài liệu preview để retry/khởi động lại giữ nguyên thời gian và payload. Chỉ xác nhận in thêm bản mới mới xóa thời gian cũ và tạo thời gian mới.
*   Hướng dẫn vận hành và giới hạn quyền của bridge tại [Windows print bridge](../tools/windows-print-bridge/README.md).