# TÀI LIỆU QUY TẮC NGHIỆP VỤ & BUSINESS LOGIC
**Mục tiêu:** Định nghĩa rõ ràng các luồng hoạt động kinh doanh, phân quyền và chuẩn mực kế toán. AI Agent bắt buộc phải code theo các rào cản logic này.

---

## 1. PHÂN QUYỀN (ROLE-BASED ACCESS CONTROL - RBAC)
Hệ thống sử dụng JWT Authentication với 3 vai trò (Roles) phân lập:

*   **`SuperAdmin` (Quản trị nền tảng SaaS):**
    *   Tạo/Khóa các Cửa hàng (Tenants).
    *   Cấu hình `Global Feature Flags` (Bật/tắt module Kho, Chấm công cho toàn hệ thống).
*   **`TenantAdmin` (Chủ cửa hàng):**
    *   Full quyền CRUD đối với dữ liệu thuộc `TenantId` của mình.
    *   Đổi giá hàng loạt (Bulk Update).
    *   Duyệt thực hiện Bút toán đảo ngược (Hủy/Sửa hóa đơn).
    *   Cấu hình `Tenant Feature Flags` (Tự động làm tròn tiền, In Bluetooth).
*   **`Staff` (Nhân viên/Thu ngân):**
    *   Chỉ được: Lập hóa đơn mới, tìm kiếm khách hàng/sản phẩm.
    *   KHÔNG được: Sửa giá gốc sản phẩm, Hủy hóa đơn đã Sync (muốn hủy phải có PIN của Admin).

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