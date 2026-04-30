# Model: Payment

## Breadcrumb

[billing-service](../../ATLAS.md) › [Models](.) › Payment

## Defined In

`internal/payment/model.go:14`

## Shape

```go
type Payment struct {
    ID                string    `db:"id"`                 // uuid, pk
    CustomerID        string    `db:"customer_id"`        // uuid
    InvoiceID         string    `db:"invoice_id"`         // uuid
    AmountCents       int64     `db:"amount_cents"`
    Currency          string    `db:"currency"`           // ISO 4217
    ProviderReference string    `db:"provider_reference"` // unique
    Status            string    `db:"status"`             // recorded | refunded | partial
    CreatedAt         time.Time `db:"created_at"`
}
```

## Used By

- [issue-refund](../issue-refund/feature.md) — reads to validate eligibility
- [list-invoices](../list-invoices/feature.md) — embeds as `last_payment`
- [record-payment](../record-payment/feature.md) — writes new rows

## Persistence

Postgres `payments` table. Unique index on `provider_reference` (idempotency). FK from `refunds.original_payment_id`.

## Last Verified

a3bd2e3c9c3c17494a4ea7fff79478d8e38ca5cc (2026-05-01)
