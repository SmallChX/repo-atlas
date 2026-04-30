# Feature Index — billing-service

## Features by Domain

### Domain: billing

- [issue-refund](issue-refund/feature.md) — Reverse a recorded payment and emit a refund event for downstream reconciliation.
- [list-invoices](list-invoices/feature.md) — Paginated invoice listing for a customer, joined with payment status.
- [record-payment](record-payment/feature.md) — Persist an inbound payment, publish `payments.recorded`, and idempotently dedupe by provider reference.

### Domain: ops

_None._

## Cross-References

- [Glossary](glossary.md)
