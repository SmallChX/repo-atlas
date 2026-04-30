# Feature: Issue Refund

## Breadcrumb

[billing-service](../../ATLAS.md) › Issue Refund

## Domain

billing

## Summary

Reverses a previously-recorded payment by inserting a row into `refunds` and emitting `payments.refunded`. Refuses if the original payment is older than 90 days or already fully refunded. Always traceable to the source payment via `original_payment_id`.

## Triggers

- `POST /v1/payments/{id}/refund` — `internal/refund/handler.go:19`
- CLI `billingctl refund <payment-id>` — `cmd/billingctl/refund.go:12`

## Public Interface

`internal/refund/handler.go:19`

```go
type IssueRefundRequest struct {
    Reason      string `json:"reason"       validate:"required,max=200"`
    AmountCents int64  `json:"amount_cents" validate:"required,min=1"`
}

func (h *Handler) IssueRefund(c *gin.Context) { ... }
```

## Models

- [payment](../_models/payment.md) — read to validate eligibility

## Code Paths

- `internal/refund/handler.go:19` — HTTP handler
- `internal/refund/service.go:37` — eligibility rules (90-day window, partial-refund check)
- `internal/refund/repo.go:44` — `INSERT INTO refunds (...) RETURNING id`
- `internal/refund/publisher.go:21` — publishes `payments.refunded`
- `db/migrations/0011_refunds.sql:1` — `refunds` table + FK to `payments.id`

## External References

- [reconciliation-worker/.atlas/payments-refunded-handler/feature.md](../../../reconciliation-worker/.atlas/payments-refunded-handler/feature.md) — downstream consumer

## See Also

- [record-payment](../record-payment/feature.md) — produces the rows refunded here
- [list-invoices](../list-invoices/feature.md) — surfaces refund status in invoice rows

## Last Verified

a3bd2e3c9c3c17494a4ea7fff79478d8e38ca5cc (2026-05-01)
