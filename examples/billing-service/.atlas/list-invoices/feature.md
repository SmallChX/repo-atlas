# Feature: List Invoices

## Breadcrumb

[billing-service](../../ATLAS.md) › List Invoices

## Domain

billing

## Summary

Returns a paginated list of invoices for a customer, each row joined with its current payment + refund status. Read-only; cursor-paginated; cached for 30 s per customer.

## Triggers

- `GET /v1/customers/{id}/invoices?cursor=...` — `internal/invoice/handler.go:22`

## Public Interface

`internal/invoice/handler.go:22`

```go
type ListInvoicesResponse struct {
    Items      []InvoiceRow `json:"items"`
    NextCursor string       `json:"next_cursor,omitempty"`
}

type InvoiceRow struct {
    InvoiceID    string  `json:"invoice_id"`
    AmountCents  int64   `json:"amount_cents"`
    Status       string  `json:"status"` // open | paid | refunded | partial
    LastPayment  *Payment `json:"last_payment,omitempty"`
}
```

## Models

- [payment](../_models/payment.md) — embedded as `last_payment`

## Code Paths

- `internal/invoice/handler.go:22` — HTTP handler, cursor parsing
- `internal/invoice/service.go:31` — status derivation (paid/refunded/partial)
- `internal/invoice/repo.go:48` — paginated SQL with `LATERAL` join on latest payment
- `internal/invoice/cache.go:17` — 30-second per-customer cache
- `db/migrations/0003_invoices.sql:1` — `invoices` table

## External References

- [api-gateway/.atlas/invoices-route/feature.md](../../../api-gateway/.atlas/invoices-route/feature.md) — upstream caller

## See Also

- [record-payment](../record-payment/feature.md) — writes the rows joined here
- [issue-refund](../issue-refund/feature.md) — affects status field

## Last Verified

a3bd2e3c9c3c17494a4ea7fff79478d8e38ca5cc (2026-05-01)
