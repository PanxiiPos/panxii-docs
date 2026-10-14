# FRONTEND DESIGN SYSTEM

## 1. Nguồn sự thật

Design system dùng `flex_color_scheme` để tạo `ColorScheme`; `hugeicons` cho icon; `forui` cho Dropdown, Dialog, DateRangePicker, Badge và Toast. Widget không tự khai báo màu/icon/component thay thế.

## 2. Token

| Nhóm | Token bắt buộc |
|---|---|
| Color | primary, onPrimary, surface, surfaceContainer, onSurface, outline, error, neutral |
| Spacing | 4, 8, 12, 16, 24, 32 |
| Radius | small, medium, large theo theme app |
| Typography | display, headline, title, body, label |
| Motion | duration fast/normal/slow, standard curve |
| Elevation | flat, raised, dialog |

Mọi widget dùng token từ `Theme.of(context)`, không dùng hex/radius/spacing magic number rải rác.

### 2.1. Semantic token trạng thái mạng và đồng bộ

Khai báo các token trong `AppThemeExtension` và cung cấp giá trị cho cả light/dark theme:

| Token | Trạng thái |
|---|---|
| `networkOnline` | Thiết bị có kết nối mạng |
| `networkOffline` | Thiết bị mất kết nối mạng |
| `syncPending` | Dữ liệu đã lưu local, chờ gửi |
| `syncUploading` | Đang gửi dữ liệu lên server |
| `syncQueued` | Server/broker đã nhận lệnh, đang chờ xử lý |
| `syncDownloading` | Đang nhận/pull dữ liệu từ server |
| `syncComplete` | Server đã xác nhận đồng bộ hoàn tất |
| `syncError` | Đồng bộ thất bại hoặc bị từ chối |

Mỗi token trạng thái phải có màu riêng, đủ tương phản trên nền sử dụng và nhất quán giữa các feature; kiểm thử cả light/dark. Không tự gán màu trong widget. Luôn đi kèm nhãn hoặc icon có semantic label; màu không phải tín hiệu duy nhất. `syncQueued` không được hiển thị như `syncComplete`.

## 3. Component states

Mỗi component cần xác định `default`, `hover/focus`, `pressed`, `disabled`, `loading`, `error`, `empty` khi phù hợp. Layout phải giữ kích thước ổn định khi state đổi.

## 4. Component catalog

- `AppPrimaryButton`, `AppSecondaryButton`, `AppDestructiveButton`
- `AppTextField`, `AppSearchField`, `AppMoneyField`
- `AppScaffold`, `AppSection`, `AppSurface`
- `AppLoading`, `AppEmptyState`, `AppErrorState`
- `AppDropdown` -> forui
- `AppDialog` -> forui
- `AppDateRangePicker` -> forui
- `AppBadge` -> forui
- `AppToast` -> forui
- `AppIconButton` -> hugeicons + semantic label

## 5. Accessibility

Hit target tối thiểu 44x44, icon-only có tooltip/semantic label, text không dùng màu làm tín hiệu duy nhất, focus visible và contrast được kiểm tra.

## 6. Review checklist

- Không có `Colors.*` trực tiếp trong feature.
- Không có `Icons.*`; chỉ `hugeicons` qua `AppIcons`.
- Không tự tạo Dropdown/Dialog/Toast khi forui đáp ứng.
- Có light/dark theme test.
- Có test tiếng Việt/Anh và màn hình nhỏ.
