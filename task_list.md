# TODO List - Action Items

## 1. Fix TenantContext Lifetime (High Priority)
- [ ] Update `DependencyInjection.cs`: Change `TenantContext` registration from **Singleton** to **Scoped**:
  ```csharp
  services.AddScoped<TenantContext>();
  services.AddScoped<ITenantContext>(
      provider => provider.GetRequiredService<TenantContext>());
  ```
- [ ] Verify that no Singleton services inject `ITenantContext` (prevent captive dependency).

## 2. Implement Auth & Tenant Middleware (High Priority)
- [ ] Create `TenantResolverMiddleware` to extract `TenantId` from JWT claim or `X-Tenant-Id` header and execute `SetTenant()`.
- [ ] Update `Program.cs`:
  - Add JWT Authentication services (`AddAuthentication`).
  - Add `app.UseAuthentication()` before `app.UseAuthorization()`.
  - Add `app.UseMiddleware<TenantResolverMiddleware>()`.
- [ ] Add `[Authorize]` attribute to `ProductsController` and related endpoints.

## 3. Support Unicode / Vietnamese Text in Database
- [ ] Update EF Core Entity Configuration to set text columns (`Name`, `Description`, etc.) to `nvarchar` (SQL Server) or ensure `UTF-8` encoding / `Vietnamese_CI_AS` collation.
- [ ] Add EF Core Migration (`Add_Unicode_Support_For_Products`) and apply to Database.
- [ ] Test CRUD operations with Vietnamese accented characters (e.g., *"Sản phẩm mẫu có dấu"*).

## 4. Refactor Error Handling & Result Pattern (Medium Priority)
- [ ] Implement `Result<T>` pattern in Application layer.
- [ ] Refactor `ProductService.cs` to return `Result<T>` instead of throwing validation exceptions.
- [ ] Remove repetitive `try-catch` blocks in `ProductsController.cs` and implement a Global Exception Handling Middleware.