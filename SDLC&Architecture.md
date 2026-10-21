# KIẾN TRÚC HỆ THỐNG VÀ QUY TRÌNH PHÁT TRIỂN (MASTER PLAN)
**Định hướng:** SaaS-Ready (Multi-tenant), Offline-First, Đa ngôn ngữ (i18n).

---

## 1. TECH STACK CHÍNH
*   **Backend:** ASP.NET Core 8 Web API, Clean Architecture.
*   **Database:** MySQL 8.0, Entity Framework Core 8 (Code-First).
*   **Mobile:** Flutter, BLoC Architecture, `easy_localization` (Đa ngôn ngữ).
*   **Local Storage:** SQLite (Mã hóa bằng `sqflite_sqlcipher`).
*   **Real-time:** SignalR (Đồng bộ real-time giữa các client).

---

## 2. DATABASE SCHEMA CỐT LÕI (SAAS)
Mọi bảng (trừ hệ thống gốc) phải thỏa mãn 3 điều kiện:
1.  Kế thừa `IMustHaveTenant` (Có cột `TenantId`).
2.  Kế thừa `IAuditable` (Có cột `CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy`).
3.  Entity phù hợp có `ExtraDataJson` kiểu `JSON` của MySQL để lưu thuộc tính nghiệp vụ bổ sung theo contract `schemaVersion`/`attributes`; không dùng thay cho cột nghiệp vụ cố định hoặc cấu hình trong `AppSettings`. Chi tiết tại [database_design.md](database/database_design.md).

**Các Entity chính:**
*   `Tenants`: Lưu thông tin cửa hàng.
*   `Users`, `UserDevices` (Quản lý session & Kill-switch).
*   `Categories`, `Products`.
*   `Customers`.
*   `Invoices`, `InvoiceDetails`.
*   `AppSettings` (Quản lý Feature Flags).

---

## 3. BẢO MẬT & DEVSECOPS
*   **Cách ly dữ liệu (Data Isolation):** Áp dụng cấu hình `Global Query Filter` của EF Core. Mọi truy vấn DB tự động gắn thêm `WHERE TenantId = @currentTenant`.
*   **Mã hóa Local:** SQLite trên điện thoại được mã hóa chuẩn AES-256 (SQLCipher).
*   **Remote Kill-Switch:** API quản lý trạng thái token. Nếu phát hiện rủi ro, Admin set `IsRevoked = true`. Mobile gọi API nhận 401 sẽ tự động **xóa file SQLite** và force logout.
*   **Network:** Đóng kín Port MySQL trên VPS, chỉ mở Port Nginx (80/443).

---

## 4. CHIẾN LƯỢC MỞ RỘNG (EXTENSIBILITY)
*   **OTA Update (Shorebird):** Sửa lỗi logic, đổi UI trên Flutter và push thẳng xuống điện thoại người dùng không qua App Store/Google Play.
*   **Force Update API:** API `GET /config/version`. Bắt buộc người dùng tải file APK/IPA mới nếu phiên bản hiện tại (Native code) quá cũ.

---

## 5. QUY TRÌNH SDLC (6 GIAI ĐOẠN)
1.  **Phân tích:** Chốt các Business Rules (Xem file 3).
2.  **Thiết kế:** Dựng UI 1 cột tối giản. Chốt cấu trúc Clean Arch.
3.  **Lập trình:** EF Core ExecuteUpdateAsync (Bulk update), SignalR Hubs.
4.  **Kiểm thử:** Unit Test (Bút toán đảo, Phân quyền), Load Test (k6 cho API Sync).
5.  **Triển khai:** Github Actions -> VPS Docker -> Nginx SSL.
6.  **Bảo trì:** Dozzle (Log), Cronjob (Backup), Shorebird (Patch OTA).