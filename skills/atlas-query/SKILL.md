---
name: atlas-query
description: Answer routing, impact-analysis, bug-triage, and cross-repo trace questions using one or more Repo Atlases without reading source code unless the atlas is insufficient. Read-only.
when_to_use: |
  Trigger phrases:
  - "where does X live?"
  - "what breaks if I change Y?"
  - "trace this contract across repos"
  - "where to look for bug Z?"

  Use this skill any time an answer can plausibly be derived from atlas content alone — routing ("which repo owns X?"), impact analysis ("what breaks if Y changes?"), bug triage ("where to start looking?"), or cross-repo trace ("who consumes this collection / topic / library?"). Auto-activates when atlas files are open. Falls back to source only when the atlas has a gap, and records the gap as a refresh signal.

  Do NOT use this skill to modify the atlas — it's read-only. Use atlas-refresh for fixes; atlas-validate for audits.
allowed-tools: read_file, list_dir, file_search, grep_search, semantic_search
paths: [".atlas/**/*.md", "ATLAS.md", "atlas.workspace.md"]
---

# Skill: atlas-query

> Read-side companion to [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md), [`atlas-bootstrap-workspace`](../atlas-bootstrap-workspace/SKILL.md), and [`atlas-refresh`](../atlas-refresh/SKILL.md). The atlas exists to answer questions cheaply; this skill is the playbook.

## Atlas-first reading order

When a question can plausibly be answered from atlas content, traverse the tree top-down and load the smallest set of files needed:

```
atlas.workspace.md  →  ATLAS.md  →  .atlas/INDEX.md  →  .atlas/<feature>/feature.md
                                                    →  .atlas/_models/<m>.md           (only when feature links there)
                                                    →  .atlas/_entrypoints/<cat>.md    (only for "list every X" queries)
```

There are no domain READMEs, no chapter `INDEX.md` files, no module/entrypoint/bridge leaves. The whole atlas is **cover + index + glossary + per-feature leaves + two flat escape hatches**. Open each level only after reading the previous one. Stop the moment the question is answered.

### Tree-traversal budget

A typical answer should load:

- **1** workspace manifest (only for cross-repo questions).
- **1** `ATLAS.md` per candidate repo.
- **1** `.atlas/INDEX.md`.
- **1–5** feature leaves.
- **0–2** shared-model leaves (only when the question is model-shaped and the model crosses ≥3 features).

If you find yourself loading every feature in the repo, the question is too coarse — reframe.

## Recipe 1 — Feature routing

> "Add feature X" — which repo(s)?

1. Open `atlas.workspace.md`. Inspect `## Domain Map`: does any tag match the feature's subject (pricing, identity, notifications)?
2. Shortlist repo(s) owning matching tag(s). One match → strong candidate. Multiple → multi-repo feature.
3. For each candidate, open `.atlas/INDEX.md` and scan `## Features by Domain` under the matching `### Domain: <tag>` heading. Look for an existing or adjacent feature.
4. If found → that repo owns the new feature. If absent but the tag fits → still that repo, with a note that this is a green-field feature.
5. If no tag matches → escalate: the workspace lacks coverage; propose adding a tag.

### Worked example — single repo

User: "Add a discount-code feature."

- `## Domain Map`: `pricing → example-billing-service`.
- `example-billing-service/.atlas/INDEX.md` `### Domain: pricing`: contains `coupon-redemption.md`. Adjacent feature confirms the tag.
- **Answer**: `example-billing-service`, tag `pricing`. Cite [atlas.workspace.md#domain-map](atlas.workspace.md) and [.atlas/INDEX.md](example-billing-service/.atlas/INDEX.md).

### Worked example — multi-repo

User: "Add SSO login."

- `## Domain Map`: `identity → example-customer-service`, `notifications → example-notification-service`.
- `example-customer-service` owns the auth flow (primary).
- `example-notification-service` owns invite emails (secondary).
- **Answer**: primary `example-customer-service` (identity); secondary `example-notification-service` (invite emails).

## Recipe 2 — Impact analysis

> "What breaks if we change X?" — forward-tracing.

1. Locate the starting entity. If a model used by ≥3 features: open `_models/<m>.md` and read `## Used By`. If a model used by ≤2 features: grep `## Models` across feature leaves for the inline mention. If a feature: open its leaf and read its `## Code Paths`, `## Triggers`, `## Models`, `## See Also`.
2. For every intra-repo `## See Also` link, follow into the linked feature (bounded — usually 2 hops is enough).
3. For cross-repo impact: open `## External References`. Bridge-anchor links land in `atlas.workspace.md`'s `## Bridges` section; direct-feature links land in another repo's feature leaf.
4. For workspace-wide cross-cutting impact: open `atlas.workspace.md` `## Cross-Repo Dependencies`.

### Worked example — model change with ≥3 consumers

User: "What breaks if we change the `Invoice` schema in `example-billing-service`?"

- `Invoice` is referenced by 4 features → it lives at `example-billing-service/.atlas/_models/invoice.md`.
- Open the leaf. `## Used By` lists `refund-flow`, `refund-engine`, `nightly-invoice-close`, `invoice-pdf-render`.
- Open `example-billing-service/.atlas/<refund-flow>/feature.md`. `## External References` cites bridge `invoice-paid-event` → `[invoice-paid-event](../../atlas.workspace.md#bridge-invoice-paid-event)`.
- Open `atlas.workspace.md` and jump to `### Bridge: invoice-paid-event`. `Consumed by:` lists `example-notification-service` — that's the cross-repo ripple.
- **Answer**:
  - Internal: `refund-flow`, `refund-engine`, `nightly-invoice-close`, `invoice-pdf-render` in `example-billing-service`.
  - External: `example-notification-service` consumes `invoice.paid` events via the `invoice-paid-event` bridge.
  - Cite each feature, the shared-model leaf, and the bridge anchor.

### Worked example — model change with ≤2 consumers

User: "What breaks if we change the `RefundSlip` shape?"

- Open `example-billing-service/.atlas/INDEX.md`. Scan for "refund". Open `<refund-flow>/feature.md`.
- `## Models` has `- RefundSlip — { id, invoice_id, amount, reason, created_at }` inlined (no link out).
- The model is therefore referenced by ≤2 features. Grep `## Models` across this repo's feature leaves for `RefundSlip` to find the other consumer (if any).
- **Answer**: changes to `RefundSlip` ripple to the 1–2 features that inline it; no shared-model leaf to update.

## Recipe 3 — Bug triage

> "Bug Y reproduces with symptom S" — where to look?

Map the symptom to the closest atlas entry:

| Symptom kind                            | Map via                                                                                |
|-----------------------------------------|----------------------------------------------------------------------------------------|
| HTTP error / endpoint misbehavior       | If catalog exists: `_entrypoints/http-routes.md` → owning feature. Else: `INDEX.md`.   |
| CLI failure                             | If catalog exists: `_entrypoints/cli-commands.md` → owning feature. Else: `INDEX.md`.  |
| Job did not run / wrong output          | `INDEX.md` features whose `## Triggers` mention a schedule or queue                   |
| Event not consumed / wrong fan-out      | `INDEX.md` features whose `## Triggers` mention a topic                                |
| Wrong business outcome (e.g., bad tax)  | `INDEX.md` features matching the user's noun (tax, invoice, refund)                    |
| Wrong stored value                      | `_models/` directory listing OR features whose `## Models` mention the entity          |

From the matching feature leaf, descend through its `## Code Paths` to produce a ranked investigation list.

### Worked example — HTTP symptom

User: "POST /refunds returns 500 in production."

- Workspace `## Domain Map`: refunds live in `billing` → `example-billing-service`.
- Open `example-billing-service/.atlas/_entrypoints/http-routes.md` (catalog exists — repo has 14 routes). Find row `- POST /refunds — issue refund → [refund-flow](../refund-flow/feature.md)`.
- Open `<refund-flow>/feature.md`. `## Triggers` cites `src/billing/api/refunds.controller.ts:12`. `## Public Interface` shows the handler signature. `## Code Paths` lists registration + dispatch + core logic + validation + persistence.
- **Investigation list** (ranked):
  1. `src/billing/api/refunds.controller.ts:12` (handler — start here).
  2. `src/billing/refund/engine.ts:45` (core dispatch).
  3. `src/billing/persistence/refund_writer.ts:88` (persistence).
- Cite the catalog row and the feature leaf.

### Worked example — domain symptom (no catalog)

User: "Customer says invoices show wrong tax."

- `## Domain Map`: `billing` → `example-billing-service`.
- Open `example-billing-service/.atlas/INDEX.md` `### Domain: billing`. Scan for tax. Find `invoice-tax-calculation`.
- Open `<invoice-tax-calculation>/feature.md`. `## Code Paths` → `src/billing/tax/calculator.ts:applyJurisdictionTax`.
- **Investigation list** (ranked):
  1. `src/billing/tax/calculator.ts:applyJurisdictionTax`.
  2. `tax-calculator` test files.
  3. The `Invoice` model's `## Persistence` (whether `_models/invoice.md` exists or it's inlined here).

## Recipe 4 — Cross-repo trace

> "Who consumes this collection / topic / library?" — bridge-driven trace via the workspace knowledge graph.

Every bridge collapses to one anchored sub-section under `## Bridges` in `atlas.workspace.md`. Per-repo bridge directories do not exist.

### Procedure

1. Open `atlas.workspace.md` `## Knowledge Graph`. Find the bullet matching the contract you're tracing (collection name, topic name, library name, gRPC method). The bullet is `- [<bridge>](#bridge-<bridge>) — <kind>: <producer> → <consumer-1>[, ...]`.
2. Click the anchor → land at `### Bridge: <name>`. Read `Contract:` for the wire shape; read `Consumed by:` for the per-consumer purpose; read `Last Verified:` for the SHA pair.
3. To find the consumer-side call site, open each consumer repo's relevant feature (its `## External References` carries a back-link to the same anchor). Descend into its `## Code Paths` for the actual file:symbol.

### Hop budget

A typical cross-repo trace should load:

- **1** `atlas.workspace.md`.
- **1 per consumer** consumer-side feature carrying the `## External References` back-link.

If you open unrelated leaves to confirm an edge, the bridge is under-documented — either `Consumed by:` is incomplete, or back-links are missing. Record the gap and recommend `atlas-refresh`.

### Worked example — binary-drop

User: "Who consumes `libfoo` from `build-foo`?"

- `atlas.workspace.md` `## Knowledge Graph`:
  > `- [libfoo-prebuilt](#bridge-libfoo-prebuilt) — binary-drop: build-foo → platform-linux, platform-macos`
- Jump to `### Bridge: libfoo-prebuilt`:
  - `Producer: build-foo/build/release.sh`
  - `Consumed by:`
    - `platform-linux` — vendor script ingests `libfoo.so` into `3rdparty/`
    - `platform-macos` — vendor script ingests `libfoo.dylib` into `3rdparty/`
  - `Contract: shared library exposing the C ABI in build-foo/include/foo.h. Release builds only.`
- Open `platform-linux/.atlas/<3rdparty-vendor>/feature.md`. `## External References` confirms the back-link to `#bridge-libfoo-prebuilt`. `## Code Paths` points at the vendor script.
- **Answer**: producer `build-foo`; consumers `platform-linux` (vendor script) and `platform-macos` (same). Cite `atlas.workspace.md#bridge-libfoo-prebuilt` and each consumer feature.

### Worked example — shared-collection

User: "Who reads or writes the `references` collection?"

- `atlas.workspace.md` `## Knowledge Graph`:
  > `- [references-collection](#bridge-references-collection) — shared-collection: content-service → search-service, analytics-service`
- Jump to `### Bridge: references-collection`:
  - `Producer: content-service/src/persistence/references_writer.go:WriteReference`.
  - `Consumed by:`
    - `search-service` — reads `references` nightly to rebuild the search index
    - `analytics-service` — reads `references` to compute citation counts
  - `Contract: MongoDB collection with documents { _id, source_id, target_id, relation, created_at }.`
- Open `search-service/.atlas/<references-importer>/feature.md`. `## External References` confirms the back-link, kind `shared-collection`. `## Code Paths` → `internal/references/importer.go:Importer.Run`.
- Same pattern in `analytics-service/.atlas/<citation-count-job>/feature.md`.
- **Answer**: producer `content-service`; consumers `search-service/references-importer` (nightly) and `analytics-service/citation-count-job`. Cite the bridge anchor + each consumer feature.

## Citation discipline

Every claim derived from atlas content MUST be accompanied by a link.

- **Same-repo claim** — relative path:
  ```
  <relative-path>     or     <relative-path>#<anchor>
  ```
- **Cross-repo claim** — cross-repo path form (matches `## External References`):
  - For bridges: `<rel>/atlas.workspace.md#bridge-<bridge-name>`
  - For direct features: `<repo-name>/.atlas/<feature>/feature.md`

A grounded answer looks like:

> The refund flow lives in `example-billing-service`. Handler is `src/billing/api/refunds.controller.ts:RefundsController.create` ([refund-flow/feature.md](example-billing-service/.atlas/refund-flow/feature.md)). Tax events are consumed by `example-notification-service` via the `invoice-paid-event` bridge ([atlas.workspace.md#bridge-invoice-paid-event](atlas.workspace.md#bridge-invoice-paid-event)).

If a sentence cannot be cited, either it doesn't belong (you're guessing) or you need to fall back to source.

## Staleness handling

Before relying on any atlas, check `## Last Verified` on the cover and the relevant feature leaf.

- **Threshold**: if `## Last Verified` is more than **20 commits behind `HEAD`** on the source repo, prefix your answer with a warning and recommend a refresh.
- **Wording**:
  > ⚠️ The atlas at `<path>` was last verified at `<sha-short>` (`<date>`) and may be stale. Run `atlas-refresh` for higher confidence. Provisional answer follows.

## Source-fallback rule

If the atlas does not contain enough detail to answer with confidence:

1. **Record the gap**:
   > _Atlas gap: no feature found for `<name>` (tag `<domain>`). Falling back to source._
2. **Widen to source**.
3. **Suggest a refresh**.

The "atlas gap" notes are the signal for the next refresh cycle.

## Layout fork

Before resolving any cross-repo path (Recipe 4), read `## Layout` from `<workspace>/atlas.workspace.md` (default `repo-rooted`).

| Layout              | Direct-feature citation form                              |
|---------------------|-----------------------------------------------------------|
| `repo-rooted`       | `<repo-name>/.atlas/<feature>/feature.md`                 |
| `workspace-rooted`  | `.atlas/<repo-name>/<feature>/feature.md` (workspace-rel) |

Bridge-anchor citations (`<rel>/atlas.workspace.md#bridge-<name>`) are layout-agnostic.

## See also

- Generation: [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md)
- Refresh: [`atlas-refresh`](../atlas-refresh/SKILL.md)
- Validate: [`atlas-validate`](../atlas-validate/SKILL.md)
