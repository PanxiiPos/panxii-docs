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
    *   Xem/tìm kiếm khách hàng và lấy thông tin khách hàng phục vụ lập hóa đơn; không được tạo, sửa hoặc xóa khách hàng.
    *   Lập hóa đơn mới.
    *   Được in hóa đơn trên thiết bị client; Bluetooth phụ thuộc cấu hình cửa hàng/feature flag.
    *   Không được quản lý tài khoản, cấu hình cửa hàng/feature flag hoặc thay đổi dữ liệu sản phẩm/khách hàng.

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