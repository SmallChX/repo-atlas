# Glossary — billing-service

### Term: Invoice

A billing artifact issued to a customer. One invoice may be settled by zero or more `Payment`s.

### Term: Payment

A recorded transfer of funds from a customer, identified by a provider reference (Stripe charge id, bank wire id, etc.). See [`_models/payment.md`](_models/payment.md).

### Term: Refund

A reversal of a previously-recorded `Payment`. Always traceable to the original payment via `original_payment_id`.

### Term: Idempotency key

The `provider_reference` field on a payment. Two requests with the same reference resolve to the same row; the second is a no-op.
