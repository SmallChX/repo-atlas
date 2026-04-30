# Feature: Record Payment

## Breadcrumb

[billing-service](../../ATLAS.md) › Record Payment

## Domain

billing

## Summary

Accepts an inbound payment from the API gateway, persists it to `payments`, and publishes a `payments.recorded` event for downstream reconciliation. Idempotent on `provider_reference` — duplicate POSTs return the original row without re-publishing.

## Triggers

- `POST /v1/payments` — `internal/payment/handler.go:24`
- NATS consumer `payments.retry` (DLQ replay) — `internal/payment/consumer.go:18`

## Public Interface

`internal/payment/handler.go:24`

```go
type RecordPaymentRequest struct {
    CustomerID        string  `json:"customer_id"        validate:"required,uuid"`
    InvoiceID         string  `json:"invoice_id"         validate:"required,uuid"`
    AmountCents       int64   `json:"amount_cents"       validate:"required,min=1"`
    Currency          string  `json:"currency"           validate:"required,iso4217"`
    ProviderReference string  `json:"provider_reference" validate:"required"`
}

func (h *Handler) RecordPayment(c *gin.Context) { ... }
```

## Models

- [payment](../_models/payment.md) — canonical persisted shape

## Code Paths

- `internal/payment/handler.go:24` — HTTP handler, request validation
- `internal/payment/service.go:41` — idempotency check + insert
- `internal/payment/repo.go:58` — `INSERT ... ON CONFLICT (provider_reference) DO NOTHING RETURNING id`
- `internal/payment/publisher.go:33` — publishes `payments.recorded` to NATS
- `internal/payment/consumer.go:18` — DLQ replay path
- `db/migrations/0007_payments.sql:1` — `payments` table + unique index on `provider_reference`

## External References

- [api-gateway/.atlas/payments-route/feature.md](../../../api-gateway/.atlas/payments-route/feature.md) — upstream caller
- [reconciliation-worker/.atlas/payments-recorded-handler/feature.md](../../../reconciliation-worker/.atlas/payments-recorded-handler/feature.md) — downstream consumer of `payments.recorded`

## See Also

- [issue-refund](../issue-refund/feature.md) — operates on rows this feature creates
- [list-invoices](../list-invoices/feature.md) — joins on `payment_id`

## Last Verified

a3bd2e3c9c3c17494a4ea7fff79478d8e38ca5cc (2026-05-01)
