# billing-service Atlas

> AI-readable map of `billing-service`. One-screen home; chapter detail lives in `.atlas/`.

## Summary

Internal billing service that records customer payments, lists invoices, and processes refunds against the central `payments` collection. Sits behind the API gateway; consumed by the ops dashboard and the support tooling.

## Domain & Purpose

Owns the **billing** bounded context for the platform. In scope: payment recording, invoice listing, refunds. Out of scope: pricing, tax calculation, subscription lifecycle (those live in `pricing-service` and `subscriptions-service`).

## Tech Stack

- Language: Go 1.22
- Framework: Gin (HTTP), sqlx (Postgres)
- Storage: Postgres 15 (`payments`, `invoices`, `refunds` tables)
- Queue: NATS JetStream (`payments.recorded` topic)

## Owners

- @platform-billing

## Domains

- billing — payment recording, invoice listing, refunds
- ops — operational endpoints (health, metrics, admin actions)

## Index

- [Feature index](.atlas/INDEX.md)
- [Glossary](.atlas/glossary.md)

## Shared Models

- [payment](.atlas/_models/payment.md) — shared by issue-refund, list-invoices, record-payment

## Last Verified

a3bd2e3c9c3c17494a4ea7fff79478d8e38ca5cc (2026-05-01)
