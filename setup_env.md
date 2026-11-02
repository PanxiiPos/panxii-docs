# HƯỚNG DẪN SETUP MÔI TRƯỜNG VÀ KHỞI TẠO DỰ ÁN (PROJECT SETUP GUIDE)
**Dự án:** Hệ thống POS Bán hàng SaaS
**Mục tiêu:** Cung cấp các lệnh CLI và cấu hình chuẩn để khởi tạo Source Code, Local Môi trường, và Production DevOps.

> Dự án đã có source; không chạy lại các lệnh tạo project bên dưới lên repository
> hiện tại. Với triển khai nội bộ tiết kiệm chi phí, làm theo
> [hướng dẫn VPS thuê và Shorebird Free](infrastructure/internal_free_deployment.md).
> Hướng dẫn đó ghi rõ bước nào chưa thực hiện; hiện chưa cần đăng nhập hay deploy.

---

## 1. YÊU CẦU CÔNG CỤ (PREREQUISITES)
*   **IDE:** Cursor, Windsurf, hoặc VS Code.
*   **Backend:** .NET 8.0 SDK, Entity Framework Core CLI (`dotnet tool install --global dotnet-ef`).
*   **Mobile:** Flutter SDK (bản Stable mới nhất).
*   **DevOps:** Docker Desktop (cho Local), Git.
*   **OTA Tool:** Shorebird CLI (Cài đặt qua `shorebird.dev`).

---

## 2. KHỞI TẠO SOURCE CODE LOCAL

### 2.1. Backend (C# Clean Architecture)
Mở Terminal tại thư mục làm việc và chạy chuỗi lệnh sau để tạo khung sườn:
```bash
mkdir PosSystem && cd PosSystem
dotnet new sln -n PosSystem

# Tạo các layer
dotnet new classlib -n PosSystem.Domain
dotnet new classlib -n PosSystem.Application
dotnet new classlib -n PosSystem.Infrastructure
dotnet new webapi -n PosSystem.WebApi

# Thêm vào Solution
dotnet sln add **/*.csproj

# Thiết lập Dependency (Reference)
dotnet add PosSystem.Application/PosSystem.Application.csproj reference PosSystem.Domain/PosSystem.Domain.csproj
dotnet add PosSystem.Infrastructure/PosSystem.Infrastructure.csproj reference PosSystem.Application/PosSystem.Application.csproj
dotnet add PosSystem.WebApi/PosSystem.WebApi.csproj reference PosSystem.Infrastructure/PosSystem.Infrastructure.csproj
dotnet add PosSystem.WebApi/PosSystem.WebApi.csproj reference PosSystem.Application/PosSystem.Application.csproj
```

### 2.2. Mobile (Flutter)
Mở Terminal tại thư mục gốc `PosSystem`:
```bash
flutter create pos_mobile_app
cd pos_mobile_app

# Khởi tạo công cụ cập nhật từ xa
shorebird init

# Thêm các package lõi
flutter pub add sqflite_sqlcipher easy_localization flutter_bloc dio signalr_netcore flex_color_scheme hugeicons forui

# Tham khảo coding standard frontend:
# Document/frontend/frontend_coding_standard.md
# Document/frontend/design_system.md
```

---

## 3. CẤU HÌNH MÔI TRƯỜNG DOCKER LOCAL (DEV)
Tạo file `docker-compose.local.yml` tại thư mục gốc để lập trình viên có Database test nội bộ:
```yaml
version: '3.8'
services:
  pos-db-local:
    image: mysql:8.0
    container_name: pos_mysql_dev
    environment:
      MYSQL_DATABASE: PosDatabaseDev
      MYSQL_USER: pos_app
      MYSQL_PASSWORD: pos_dev_password_only
      MYSQL_ROOT_PASSWORD: root_dev_password_only
    ports:
      - "3306:3306"
  
  dozzle-local:
    image: amir20/dozzle:latest
    container_name: pos_dozzle_dev
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "8888:8080"
```
*Lệnh chạy:* `docker-compose -f docker-compose.local.yml up -d`

---

## 4. QUẢN LÝ BIẾN MÔI TRƯỜNG (.env)
Tạo file `.env` (BẮT BUỘC thêm vào `.gitignore`):
```env
DB_CONNECTION=Server=pos-db;Database=PosDatabase;User=pos_app;Password=UseLocalSecretOnly!
JWT_SECRET_KEY=ChuoiKyTuBiMatDaiHon256BitsChoHeThongSaaS!
```

---

## 5. TRIỂN KHAI PRODUCTION (VPS)
Không dùng cấu hình Docker local hoặc ví dụ khởi tạo cũ để deploy production.
Các bước triển khai, quản lý secret, backup và CI/CD được tập trung tại:

- [Setup nội bộ miễn phí để thực hiện sau](infrastructure/internal_free_deployment.md).
- [Hạ tầng production chuẩn](infrastructure/infrastructure_setup.md).
- [Checklist phát hành và rollback](release/release_management.md).

Chỉ Nginx mở public `80/443`; không public MySQL/RabbitMQ/internal API.
Deploy image theo Git SHA sau khi test, review migration và có backup ngoài VPS.
Không truyền mật khẩu database trên dòng lệnh hoặc ghi secret vào Git/Actions log.