# ADR-012: In A4/A5 qua PC Windows và QR chỉ dẫn chuyển khoản

- Status: Accepted
- Cập nhật 2026-11-02: mở rộng in thực tế từ A4 sang A4/A5 theo yêu cầu mới;
  UI chỉ hiện hai khổ này, ẩn nhập kích thước/lề và ngân hàng, giữ dữ liệu cũ.
- Cập nhật 2026-11-04: thông tin khách hàng nằm ở cột trái cùng hàng QR tra cứu,
  tránh để trống vùng dưới số hóa đơn; dòng đầu header phải nhấn mạnh vừa phải,
  các dòng mô tả còn lại tự xuống hàng khi dài.

## Context

Chủ cửa hàng dùng Canon LBP 2900 nối cáp PC Windows, muốn in từ điện thoại qua
Wi-Fi, cấu hình ngân hàng nhận tiền và header hai cột. Phạm vi trước đây trì hoãn
in và không có thông tin thanh toán trên bản in; yêu cầu mới cho phép QR nhận tiền,
nhưng không mở module Customer hoặc nghiệp vụ thu tiền/đối soát.

## Decision

- Mobile render preview và PNG A4/A5 cùng canvas, gửi trực tiếp qua HTTPS trong LAN
  tới script Windows; PC dùng driver đã cài. Không qua backend/Docker hoặc ESC-POS.
- HTTPS ghép nối token và SHA-256 pin; secure storage tenant/user trên điện thoại,
  ACL tài khoản Windows cho state PC. Bridge không kiểm tra JWT/role tenant; chỉ
  cấp token cho thiết bị tin cậy, không mở port ra Internet.
- TenantAdmin bật EnableWifiPrint, mặc định false; Staff có quyền đọc hóa đơn
  được in khi cờ đã xác nhận. Bluetooth vẫn giữ cấu hình, chưa triển khai.
- VietQR tạo local từ BIN/tài khoản/tổng VND; không memo/tên người chuyển theo lựa
  chọn cuối của chủ cửa hàng. Nội dung mặc định do app ngân hàng quyết định.
  QR không chứng minh đã trả tiền; không thêm payment state/settlement.
- Print UUID/hash được persist trước gửi; PC persist trước claim. Không tự in lại
  trạng thái uncertain. Submitted chỉ là Windows queue nhận job.
- Bản in bắt buộc số hóa đơn/barcode/ngày/dòng khách hàng/chữ ký. Header v2 và
  bankTransfer cần rollout backend trước mobile và chặn editor cũ ghi mất dữ liệu.
- Ngày in chỉ xuất hiện trong PNG thực gửi máy in, ngay dưới ngày tạo; preview
  không giả định thời điểm in trước khi gửi.
- Với QR tra cứu đặt bên phải, giữ số hóa đơn, ngày và các dòng khách hàng trống
  trong cột trái của cùng khối. Như vậy nội dung khách bắt đầu sát ngày tạo thay
  vì đợi toàn bộ chiều cao QR; không đổi thứ tự cột/bảng hoặc khổ giấy.
- Header dịch vụ tùy chỉnh vẫn giữ nguyên nội dung và căn giữa; dòng đầu đậm
  16 pt, phần còn lại 13 pt và wrap theo chiều rộng cột. Tên cửa hàng/header trái
  giữ 24 pt.

## Alternatives considered

- ESC-POS hoặc Bluetooth trực tiếp: không phù hợp máy in laser USB hiện có.
- In qua backend: tăng phụ thuộc mạng ngoài và đưa dữ liệu job ra khỏi LAN.
- Dịch vụ ảnh QR ngoài: không cần thiết, tăng phụ thuộc và truyền dữ liệu tài khoản.

## Consequences

PC phải chạy script, không sleep; cần cài Python/driver và cấu hình firewall
Private/LocalSubnet. Mất kết nối không rollback invoice. Cần kiểm tra bản in giấy,
QR trong banking app và barcode trên Canon thực tế trước sử dụng. Không thể đảm
bảo nội dung chuyển khoản mặc định của mọi app ngân hàng.

## Related documents

- [Business rules](../../business_rule.md)
- [API compatibility](../../api/api_compatibility_matrix.md)
- [Windows bridge](../../../tools/windows-print-bridge/README.md)
