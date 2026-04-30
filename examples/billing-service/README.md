# Example: `billing-service` atlas

> A sample atlas for a small fictional Go service — what `atlas-generate-tree` produces for a 3-feature repo. Use this as a reference when authoring or reviewing your own atlas.

```
billing-service/
  ATLAS.md                                # cover, ≤80 lines
  .atlas/
    INDEX.md                              # ## Features by Domain
    glossary.md
    issue-refund/feature.md               # 11 required sections, ≤150 lines
    list-invoices/feature.md
    record-payment/feature.md
    _models/payment.md                    # promoted: shared by ≥3 features
```

The fictional source files referenced in the leaves (`internal/refund/...`, etc.) do not exist — citations are illustrative.
