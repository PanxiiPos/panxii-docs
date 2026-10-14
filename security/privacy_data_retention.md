# PRIVACY VÀ DATA RETENTION POLICY

## 1. Phân loại dữ liệu

| Loại | Ví dụ | Bảo vệ |
|---|---|---|
| Credential | Password hash, refresh token hash | Hash/encryption, access control |
| Payment/security | PIN hash, provider transaction id | Least privilege, redaction |
| Customer PII | Phone, email, address | Tenant isolation, retention |
| Financial | Invoice/payment/audit | Immutable/restricted access |
| Technical | Logs, traces, metrics | Redaction, limited retention |

## 2. Nguyên tắc

- Chỉ thu thập field cần cho nghiệp vụ.
- SuperAdmin không mặc định xem PII tenant.
- Log không chứa token, PIN, password, full payment data hoặc PII không cần thiết.
- Backup được mã hóa và access logged.
- Xóa/ẩn PII không được phá tính toàn vẹn hóa đơn; dùng anonymization theo chính sách pháp lý.
- Mọi export customer data phải authorize, audit và rate limit.

## 3. Retention cần chốt

Các thời hạn dưới đây là retention dữ liệu phía server và phải được phê duyệt theo yêu cầu kế toán, pháp luật và consent; chúng không đồng nghĩa với thời gian lưu cache trên thiết bị. Mobile áp dụng riêng: cache invoice đã đồng bộ tối đa 30 ngày và cache customer tối đa 3 tháng. Hai thời hạn local này không áp dụng cho dữ liệu server, audit, bản nháp hoặc dữ liệu/outbox chưa đồng bộ.

| Data | Retention |
|---|---|
| Invoice/payment | Theo yêu cầu kế toán/pháp luật |
| Audit | Theo security/compliance policy |
| Customer PII | Theo mục đích và consent |
| Access/error logs | Ngắn hơn transaction data |
| Outbox/DLQ | Đủ cho replay/audit |
| Backups | Theo RPO/DR policy |

Thời hạn chính thức phải được phê duyệt trước production; không hardcode trong code nếu chưa có policy.

## 4. Rights and incidents

Cần runbook cho access/export/correction/anonymization, data breach, secret leak và tenant offboarding. Mọi incident có PII phải có owner, timeline, evidence và quyết định thông báo theo luật áp dụng.
