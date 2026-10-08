# PAYMENT SPECIFICATION

## 1. Phạm vi phase 1

Phase 1 hỗ trợ `Cash`. Các phương thức card, QR và e-wallet chỉ được bật sau khi có provider contract và đối soát.

## 2. Payment model

Một invoice có thể có một hoặc nhiều payment transaction tùy policy. Payment không lưu trong `ExtraDataJson` nếu cần báo cáo/đối soát.

| Field | Quy tắc |
|---|---|
| `PaymentId` | UUID server |
| `InvoiceId` | Invoice cùng tenant |
| `Method` | `Cash`, `Card`, `Qr`, `EWallet` |
| `Status` | `Pending`, `Authorized`, `Captured`, `Failed`, `Refunded` |
| `Amount` | Decimal currency |
| `PaidAmount` | Số tiền khách đưa, chủ yếu Cash |
| `ChangeAmount` | `PaidAmount - Amount`, không âm |
| `ProviderTransactionId` | Unique khi có provider |
| `IdempotencyKey` | Bắt buộc command payment |
| `CreatedAtUtc` | Server timestamp |

## 3. Invariants

- Invoice không được `Paid` nếu tổng payment captured chưa đủ total.
- Cash phải có `PaidAmount >= Amount`.
- Payment captured không update/delete; refund tạo transaction mới.
- Provider callback phải idempotent.
- Payment và invoice cùng tenant.
- Printer failure không đổi payment status.

## 4. Flow

```text
Invoice sync -> Payment authorize/capture -> payment transaction -> outbox -> reconciliation
```

Nếu payment provider timeout, trạng thái là `Pending`, không retry mù bằng key mới. Query payment status bằng cùng idempotency/provider transaction id.

## 5. API cần có khi bật non-cash

- `POST /invoices/{id}/payments`
- `GET /payments/{id}`
- `POST /payments/{id}/refund`
- `POST /payments/provider-callback`
- `GET /payments/reconciliation`

Mỗi API phải thêm OpenAPI schema, error code, audit, webhook signature verification và contract test.

## 6. Error codes

`ERR_PAYMENT_NOT_FOUND`, `ERR_PAYMENT_AMOUNT_INVALID`, `ERR_PAYMENT_ALREADY_CAPTURED`, `ERR_PAYMENT_PROVIDER_TIMEOUT`, `ERR_PAYMENT_SIGNATURE_INVALID`, `ERR_PAYMENT_RECONCILIATION_FAILED`.
