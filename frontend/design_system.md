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

### 4.1. Cấu hình thông báo dùng chung

Đây là quy chuẩn đích khi triển khai. Flutter hiện có `AppDialog.showNotice` và `AppFeedbackConfig` dùng cho Login; `AppDialog.confirm` dùng chung cho đăng xuất, bỏ thay đổi và tải lại form conflict. Confirmation không đóng khi chạm ngoài, mặc định focus Hủy, trả kết quả typed; chỉ kết quả xác nhận mới thực hiện hành động. Settings dùng feedback inline có semantic live region cho lỗi/thành công và progress ổn định. `AppFeedback`/`AppToast` và điều phối hàng đợi toàn cục chưa được triển khai. Popup thông báo phải đi qua wrapper chung, không cấu hình riêng tại từng màn hình. Popup chọn dữ liệu/filter vẫn thuộc component tương ứng, không dùng dialog thông báo thay cho form.

Tập trung cấu hình tại app theme và một policy dùng chung `AppFeedbackConfig`:

| Nhóm | Cấu hình tập trung |
|---|---|
| Giao diện | Semantic color theo mức độ, icon qua `AppIcons`, typography, spacing, radius, overlay và motion từ theme; hỗ trợ light/dark |
| Toast | Vị trí phía trên trong safe area, thời gian theo loại, nút đóng, giới hạn hàng đợi và chống lặp |
| Dialog | Chiều rộng responsive, nội dung cuộn, bố trí action, focus, đóng bằng Back/chạm ngoài và typed result |
| Nội dung | Title/message/action đã localization; phân biệt mức độ `info`, `success`, `warning`, `error` với cách hiển thị |

Feature chỉ cung cấp nội dung, mức độ, intent/action và khóa chống lặp khi cần. Không truyền màu, font size, duration hoặc `barrierDismissible` tùy ý. Các ngoại lệ phải thành preset có tên tại cấu hình chung và có test, không tạo nhánh cấu hình rải rác.

### 4.2. Chọn toast, dialog hay banner

Mức độ lỗi không tự động quyết định modal: chỉ dùng dialog khi cần người dùng xác nhận hoặc xử lý trước khi tiếp tục.

| Tình huống | Cách hiển thị | Quy tắc mặc định |
|---|---|---|
| Thông tin nhẹ, thao tác thành công | Toast `info`/`success` | Tự đóng sau 4 giây; có nút đóng; không chặn thao tác |
| Cảnh báo/lỗi tạm thời không chặn thao tác | Toast `warning`/`error` | Tự đóng sau 6 giây; có nút đóng; chỉ dùng nếu không cần giữ thông tin để xử lý |
| Sai dữ liệu nhập | Lỗi tại field/form | Giữ dữ liệu và lỗi tới khi sửa; không chỉ hiện toast rồi mất |
| Thông tin quan trọng cần đọc/xác nhận, lỗi chặn bước tiếp theo | Dialog thông báo | Không tự đóng; title, nội dung và action rõ ràng |
| Xóa, đảo hóa đơn, đăng xuất thiết bị hoặc mất dữ liệu | Dialog xác nhận destructive | Nêu đối tượng và hậu quả; có Hủy và action cụ thể, không dùng nhãn mơ hồ “Có/Không” |
| Mất mạng, pending/queued, đang đồng bộ hoặc lỗi sync nền | Banner/status toàn cục | Duy trì theo trạng thái thật; tap mở chi tiết; không bật modal mỗi lần retry |
| Phiên bị thu hồi, thiết bị bị revoke, bắt buộc cập nhật | Flow toàn cục theo auth/update state | Không chỉ báo toast; dừng/chuyển flow theo policy tương ứng, không chờ người dùng đóng dialog mới bảo vệ phiên |

`202 Queued` chỉ báo đang xử lý; lưu local chỉ báo đã lưu trên thiết bị. Không dùng toast thành công “Đã đồng bộ” trước khi event/pull xác nhận.

### 4.3. Đóng, xếp hàng và tương tác

- Toast không chứa action bắt buộc hoặc thông tin quan trọng chỉ xuất hiện một lần. Khi screen reader hoạt động hoặc hệ điều hành yêu cầu thêm thời gian đọc, tắt tự đóng và giữ nút đóng; không bắt người dùng đọc kịp trong 4/6 giây.
- Tối đa một toast hiển thị và ba toast chờ. Gộp sự kiện trùng theo intent/operation ID; khi đầy, bỏ toast tạm thời cũ nhất đang chờ. Không đưa thông báo cần xử lý vào hàng đợi có thể bị bỏ này.
- Tối đa một dialog hiển thị; chặn mở lặp cùng intent/operation ID. Không tự xếp hàng xác nhận nghiệp vụ để bật lại trên route/phiên không còn phù hợp; caller phải kiểm tra context, route và trạng thái mới nhất.
- Dialog mặc định không đóng khi chạm ngoài. Dialog thông báo có thể Back như action đóng; dialog xác nhận có thể Back như Hủy. Không có trường hợp Back/chạm ngoài/null result được hiểu là xác nhận.
- Focus ban đầu ở nội dung hoặc action an toàn, không tự đặt vào destructive action. Giữ focus trong modal và trả về nơi mở khi còn tồn tại; công bố title/message cho screen reader.
- Chỉ thực hiện nghiệp vụ sau typed result xác nhận. Khi đang thực thi, khóa submit lặp; nếu dialog chứa tiến trình thì không cho Back/chạm ngoài/đóng làm mất trạng thái. Lỗi phải hiển thị rõ và cho thử lại hoặc hủy khi an toàn, không kẹt loading.
- Khi route bị hủy, logout hoặc đổi phiên/tenant, dọn toast/dialog cũ; không gọi callback của phiên trước. Dialog không thay thế authorization hoặc validation nghiệp vụ.

## 5. Accessibility

Hit target tối thiểu 44x44, icon-only có tooltip/semantic label, text không dùng màu làm tín hiệu duy nhất, focus visible và contrast được kiểm tra.

### 5.1. Chữ lớn, dễ đọc

- Cỡ gốc tại mức scale 1.0: nội dung thông thường dùng `bodyLarge` tối thiểu 16 logical pixels; nội dung toast/dialog tối thiểu 18; tiêu đề dialog tối thiểu 20; nhãn action tối thiểu 16. Khai báo tại typography chung, không gán `fontSize` trong feature.
- Toast/dialog dùng typography theo semantic role từ cùng nguồn theme. Adapter `forui` phải nhận cùng font, text scaling và semantic color với app, không duy trì bộ giá trị độc lập.
- Không giới hạn nội dung quan trọng bằng ellipsis hoặc `maxLines` cố định. Dialog có vùng nội dung cuộn; action xuống dòng/xếp dọc khi thiếu chỗ và luôn tiếp cận được. Toast co giãn, không che navigation, bàn phím hoặc SyncStatus; nội dung dài cần đọc lại chuyển sang banner/dialog phù hợp.
- Không cố định chiều cao text/button. Vùng chạm tối thiểu 44x44 và tăng theo nội dung; trạng thái có thể giữ ổn định trong cùng mức chữ nhưng phải đo lại layout khi đổi cỡ chữ.

### 5.2. Cài đặt cỡ chữ toàn app

Thêm mục **Hiển thị > Cỡ chữ** trong cài đặt ứng dụng trên thiết bị, tách khỏi cấu hình cửa hàng cần quyền TenantAdmin. Cả Staff và TenantAdmin được dùng; ở màn hình đăng nhập cũng có lối vào tùy chọn hiển thị.

| Lựa chọn | Hệ số bổ sung của app |
|---|---|
| Theo hệ thống (mặc định) | 1.00 |
| Lớn | 1.15 |
| Rất lớn | 1.30 |

- Mọi lựa chọn vẫn tôn trọng text scaling của hệ điều hành, kể cả scaling phi tuyến. Hệ số app nhân lên kích thước sau khi áp dụng system `TextScaler`; “Theo hệ thống” không có nghĩa là ép scale về 1.0. Không clamp cỡ chữ để chữa lỗi layout.
- Áp dụng đúng một lần ở app root cho mọi route và overlay, bao gồm login, form, danh sách, toast, dialog và component `forui`. Không nhân thêm lần nữa trong `TextTheme` hay từng widget; icon/ảnh không tự phóng theo hệ số chữ.
- Có preview nội dung và nút trước khi chọn, đánh dấu lựa chọn hiện tại và action “Theo hệ thống” để khôi phục. Thay đổi có hiệu lực ngay, không restart, không mất nội dung form/nháp hoặc vị trí điều hướng.
- Lưu lựa chọn local theo cài đặt của thiết bị/app, dùng được offline và khôi phục sau khi mở lại app; không lưu thành StoreProfile/ProductDisplay, không sync outbox, không thêm API. Thiết bị dùng chung dùng cùng lựa chọn; không chứa PII và không ghi đè bằng cấu hình cửa hàng.
- Khi lưu thất bại, báo lỗi đã localization, giữ lựa chọn đã lưu trước đó và cho thử lại; không báo lưu thành công. Khi chưa từng có lựa chọn thì dùng “Theo hệ thống”; lỗi đọc hoặc giá trị không hợp lệ phải được ghi nhận theo error/log catalog và có thông báo dễ hiểu để người dùng đặt lại.
- Cỡ chữ này chỉ dành cho giao diện, không đổi dữ liệu tiền, payload, barcode hoặc kích thước mẫu in hóa đơn.

## 6. Review checklist

- Không có `Colors.*` trực tiếp trong feature.
- Không có `Icons.*`; chỉ `hugeicons` qua `AppIcons`.
- Không tự tạo Dropdown/Dialog/Toast khi forui đáp ứng.
- Có light/dark theme test.
- Có test tiếng Việt/Anh và màn hình nhỏ.
- Toast/dialog/banner tuân theo phân loại, duration, dismiss, chống lặp và lifecycle ở mục 4; thông tin cần xử lý không bị tự đóng hoặc bỏ khỏi hàng đợi.
- Kiểm thử màn hình rộng 320 logical pixels, vi/en, light/dark, bàn phím mở, screen reader và system text scale 100%/200% với từng lựa chọn cỡ chữ; có test system scaling phi tuyến, không scale hai lần.
- Không tràn/cắt nội dung quan trọng; action vẫn chạm/cuộn tới được. Đổi cỡ chữ khi dialog đang mở và khi form có dữ liệu không làm mất state.
- Cỡ chữ có hiệu lực trên login và mọi route/overlay; khôi phục đúng sau offline restart. Test lỗi đọc/ghi local, giá trị không hợp lệ và khả năng đặt lại “Theo hệ thống”.
