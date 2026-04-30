---
name: atlas-generate-tree
description: Write a complete atlas tree (ATLAS.md + .atlas/) for one repo from a confirmed domain-tag proposal. Performs filesystem writes; six phases with two flat escape hatches (_models/, _entrypoints/) and no per-domain chapter directories.
when_to_use: |
  Trigger phrases:
  - "generate atlas"
  - "build the atlas tree for this repo"
  - "write the .atlas/ for this repo"

  Use this skill once a domain-tag proposal exists (from atlas-identify-domains, or hand-supplied) and you need to materialise the full atlas tree. This skill writes many files in one repo.

  Do NOT use this skill on a multi-repo workspace bootstrap — use atlas-bootstrap-workspace, which orchestrates this skill for each repo. Do NOT use to refresh an existing atlas — use atlas-refresh.
allowed-tools: read_file, list_dir, file_search, grep_search, semantic_search, run_in_terminal, create_file, replace_string_in_file, multi_replace_string_in_file
---

# Skill: atlas-generate-tree

> Phase 3 of 4 in the generation family. **Writes files.** Materialises the full atlas tree for a single repo from a confirmed domain-tag proposal.

## Inputs

- **Repo path**.
- **Domain-tag proposal** (from `atlas-identify-domains`, or hand-supplied).
- **Repo name** *(optional)*: defaults to directory basename.
- **Owners** *(optional)*: from CODEOWNERS if not supplied.
- **Workspace context** *(optional)*: list of sibling-repo names plus their declared cross-repo edges, used so feature leaves can write `## External References` entries pointing at workspace-manifest bridge anchors. When invoked standalone (no workspace), `## External References` sections are populated only from intra-workspace siblings already atlassed (or omitted).

## Output

A tree at `<repo>/`:

```
ATLAS.md                                 # cover, ≤80 lines, ## Domains tag list
.atlas/
  INDEX.md                               # ## Features by Domain (grouped under ### Domain: <tag>)
  glossary.md
  dependencies.md                        # OPTIONAL — omit when no significant external deps
  <feature-1>/feature.md                 # one directory per feature, leaf ≤150 lines
  <feature-2>/feature.md
  …
  _models/<model>.md                     # OPTIONAL — only when ≥3 features share a model
  _entrypoints/http-routes.md            # OPTIONAL — only when ≥10 routes / ≥3 features
  _entrypoints/cli-commands.md           # OPTIONAL — only when ≥10 CLI subcommands
```

There SHALL be no `domains/` directory, no `modules/` directory, no `bridges/` directory. Bridges live in `<workspace>/atlas.workspace.md` (orchestrator-owned). Domain tags are `## Domain` lines inside each feature, not directories.

All file budgets in the schema spec are hard limits.

## Templates (co-located with this skill)

- [./templates/cover.md](./templates/cover.md) — `<repo>/ATLAS.md` shape.
- [./templates/INDEX.md](./templates/INDEX.md) — `.atlas/INDEX.md` shape.
- [./templates/glossary.md](./templates/glossary.md) — glossary seed.
- [./templates/feature.md](./templates/feature.md) — the 11-section feature leaf.
- [./templates/shared-model.md](./templates/shared-model.md) — escape-hatch model leaf.
- [./templates/http-routes.md](./templates/http-routes.md) — entrypoint catalog (HTTP).
- [./templates/cli-commands.md](./templates/cli-commands.md) — entrypoint catalog (CLI).

## Six-phase workflow

### Phase 1 — Cover and glossary seed

Compose `ATLAS.md` from [./templates/cover.md](./templates/cover.md). Required sections in order: `## Summary`, `## Domain & Purpose`, `## Tech Stack`, `## Owners`, `## Domains` (the tag list — one bullet per tag with one-line description, alphabetical), `## Index` (single bullet pointing at `.atlas/INDEX.md`), optionally `## Shared Models` (only when `_models/` will be non-empty after Phase 5), optionally `## Bridges Produced` (back-links to `<rel>/atlas.workspace.md#bridge-<name>` for bridges this repo produces; recommended but not required), `## Last Verified`.

Set `## Last Verified` to the survey-captured SHA + today's date.

Seed `.atlas/glossary.md` from [./templates/glossary.md](./templates/glossary.md) with domain terms encountered during survey.

### Phase 2 — INDEX.md skeleton

Write `.atlas/INDEX.md` from [./templates/INDEX.md](./templates/INDEX.md). Sections, in order: `## Features by Domain` (one `### Domain: <tag>` heading per tag, alphabetical, with body `_None._` for now — Phase 4 fills the bullets); `## Cross-References` (link to `glossary.md` and, when a workspace manifest is present, to its `## Bridges` section).

There SHALL be no `## Entity Index`, no `## Domains` (the cover owns the tag list), no chapter sections.

### Phase 3 — `dependencies.md` (optional)

Write `.atlas/dependencies.md` only if at least one of:

- The repo has ≥3 distinct domain tags AND non-trivial inter-tag dependencies.
- The repo has notable third-party services or libraries worth listing centrally.
- A workspace manifest exists and this repo participates in ≥1 cross-repo dependency.

Otherwise skip the file (its absence is not a validation finding). When written, sections in order: `## Internal Domain Graph` (edges between tags, derived from feature `## See Also`), `## External Dependencies` (third-party services + notable libraries), `## Cross-Repo Dependencies` (mirroring this repo's row in `atlas.workspace.md#cross-repo-dependencies`).

### Phase 4 — Features (the bulk of the work)

For every feature in the proposal, write `<repo>/.atlas/<feature-kebab>/feature.md` from [./templates/feature.md](./templates/feature.md). Required sections in order:

```
# Feature: <name>
<!-- provenance: <topic>: <chosen> (default | alt-N | freeform) -->     (only when interactive-confirmation resolved a decision)

## Breadcrumb
[Atlas](../../ATLAS.md) › <feature-name>

## Domain
<domain-tag>      (single kebab-case identifier matching one entry in the cover's ## Domains list)

## Summary
<2–4 sentence what-it-does, where-it-surfaces>

## Triggers
- <trigger-1: route / argv form / schedule / topic / library entry-point> — `<file>:<line>`
- <trigger-2: …> — `<file>:<line>`
  …                                  (≥1 cited bullet)

## Public Interface
<file>:<line>
```<lang>
<excerpt of canonical declaration(s) from source, ≤30 lines>
```

## Models
- <model-name> — <one-line shape>
- [<shared-model-name>](../_models/<model>.md) — <one-line role>      (link only when ≥3 features share — populated by Phase 5)

## Code Paths
- `<file>:<line>` — <one-line note>
  …                                  (≥5 cited bullets — registration, dispatch, core logic, validation, persistence)

## External References
- [<bridge-name>](../../atlas.workspace.md#bridge-<bridge>) — <kind>: <one-line purpose>
- [<feature-name>](<sibling-repo>/.atlas/<feature>/feature.md) — <one-line purpose>
                                     (or `_None._` when this repo has no cross-repo edges)

## See Also
- [<other-feature>](../<other-feature>/feature.md) — <one-line relationship>
                                     (or `_None._`; intra-repo only)

## Last Verified
<sha> (YYYY-MM-DD)
```

**Depth bar (hard minimums per leaf — fail validation otherwise):**

- `## Summary` — 2–4 sentences answering "what user-meaningful capability and where does it surface".
- `## Triggers` — ≥1 concrete trigger with a `<file>:<line>` citation. Generic descriptions are not acceptable.
- `## Public Interface` — fenced code block excerpted from source (≤30 lines) with `<file>:<line>` immediately above the fence. Trim verbosely; preserve signatures + meaningful types.
- `## Models` — every model the feature touches, inline as `- <name> — <one-line shape>`. Phase 5 may rewrite some entries as links to `_models/`.
- `## Code Paths` — ≥5 `<file>:<line>` bullets each with a one-line note. Generic paths like `core/` are NOT code paths. Citations MUST resolve in the working tree.
- `## See Also` — intra-repo links only, OR `_None._`. Cross-repo links go in `## External References`.

The leaf budget is 150 lines. If a feature would exceed the budget, this is a feature-naming-split signal — surface a confirmation per [Interactive Confirmation](#interactive-confirmation) below; do NOT silently truncate.

**Self-contained source reading**: open ≥2 source files per feature (registration site + dispatch site OR core implementation site). Directory listings are not sufficient. Do NOT guess line numbers.

### Phase 5 — `_models/` (conditional escape hatch)

After all features are written, count cross-feature model references by aggregating every feature's `## Models` section. For each distinct model name:

- **ref-count ≥ 3** → promote to a leaf at `.atlas/_models/<model-kebab>.md` using [./templates/shared-model.md](./templates/shared-model.md). In every feature that referenced the model, rewrite the inlined entry as `- [<model>](../_models/<model>.md) — <one-line role>`.
- **ref-count ≤ 2** → keep inline; do NOT create a `_models/` leaf.

Shared-model leaf shape (≤80 lines, sections in order): `# Model: <name>` → `## Breadcrumb` → `## Defined In` (`<file>:<line>`) → `## Shape` (fenced code block excerpted verbatim, ≤30 lines) → `## Used By` (relative links to every referencing feature, alphabetical) → `## Persistence` (store + table name, or `none`) → `## Last Verified`.

There SHALL be no `INDEX.md` under `_models/`. The directory listing is the index. If `_models/` is non-empty, add a `## Shared Models` section to `ATLAS.md` listing model names alphabetically with relative links.

### Phase 6 — `_entrypoints/` (conditional catalog)

Re-evaluate the entrypoint-catalog flags from the survey:

- **HTTP catalog**: write `_entrypoints/http-routes.md` from [./templates/http-routes.md](./templates/http-routes.md) iff the repo exposes ≥10 routes spread across ≥3 features. Each row: `- <method> <path> — <one-line description> → [<feature>](../<feature>/feature.md)`. Alphabetical by `<path>`.
- **CLI catalog**: write `_entrypoints/cli-commands.md` from [./templates/cli-commands.md](./templates/cli-commands.md) iff the repo exposes ≥10 CLI subcommands. Each row: `- <command> — <one-line description> → [<feature>](../<feature>/feature.md)`. Alphabetical by `<command>`.

The catalog is a **reverse-index**, not a canonical record — every row's trigger MUST also appear in its owning feature's `## Triggers` section. Below threshold, do NOT write the catalog.

### Final — INDEX.md fill, reciprocal links, last-verified, self-review

After Phases 4–6 complete:

1. **Fill `## Features by Domain` in `INDEX.md`**: under each `### Domain: <tag>` heading, one bullet per feature whose `## Domain` matches, alphabetical, in the form `- [<feature>](<feature>/feature.md) — <summary harvested from feature's ## Summary first sentence>`.
2. **Reciprocal `## See Also`**: for each `A → B` intra-repo link, open B and ensure `B → A` exists. Sort alphabetically.
3. **Set `## Last Verified`** on `ATLAS.md`, every feature leaf, every shared-model leaf, every entrypoint catalog: same SHA (full 40 chars) + today's `YYYY-MM-DD`.
4. **Self-review checklist**:
   - [ ] No `domains/`, `modules/`, `bridges/`, or per-feature `entrypoints/` directories.
   - [ ] Every feature has all 11 sections in order, ≤150 lines.
   - [ ] Every feature `## Domain` line matches an entry in the cover's `## Domains` list.
   - [ ] `_models/` exists iff at least one model has ref-count ≥3.
   - [ ] `_entrypoints/` exists iff route/command threshold is met.
   - [ ] Repo total feature count ≤50.
   - [ ] Every `<file>:<line>` citation resolves in the working tree.
5. **Hand off** to `atlas-validate`. Do not declare the run complete until validate is green.

## Redaction checklist

Before writing any leaf, scan its content for:

- Secrets (hard-coded API keys, JWT secrets, passwords, certificates) — never include the value or a transcribable hint.
- Internal hostnames not safe to share — use placeholders.
- Customer-identifying data (real IDs, emails, phone numbers) — replace with synthetic data.
- Compliance-restricted strings (HIPAA, PCI, GDPR, NDA).
- Anything excluded by the source repo's `.gitignore` or security policy.

If you find a secret while surveying code, do not write it to the atlas. Note the file path so a human can rotate it.

## Layout fork

Before any write, read `## Layout` from `<workspace>/atlas.workspace.md` if present; default to `repo-rooted` when no manifest exists or no section is set.

| Layout              | Cover write target                              | Subtree write target              |
|---------------------|-------------------------------------------------|-----------------------------------|
| `repo-rooted`       | `<repo>/ATLAS.md`                               | `<repo>/.atlas/`                  |
| `workspace-rooted`  | `<workspace>/.atlas/<repo>/ATLAS.md`            | `<workspace>/.atlas/<repo>/`      |

The six-phase workflow is otherwise identical. Cross-repo links emitted by this skill (in `## External References` and optional `dependencies.md`) MUST use the form mandated by the active layout.

Refuse to write outside `<workspace>/` in `workspace-rooted` mode and outside `<repo>/` in `repo-rooted` mode. Refuse to create any `domains/`, `modules/`, or `bridges/` directory under any layout.

## Interactive Confirmation

Surface a confirmation question only when the AI identifies **≥2 plausible alternatives**.

**Decision points** (each surfaced as a batched checkpoint at the start of the affected phase; ≤10 per batch, themed sub-batches on overflow):

1. **Per-tag feature list and feature names** *(Phase 4)* — before any feature leaf is written, when at least one proposed feature could equally plausibly be split into 2–3 narrower features (typical signal: the proposed leaf would exceed 150 lines, or the source files span ≥2 entry-point types). Default = umbrella name; alternatives = the split names with rationale.
2. **Entry-point Type assignment** *(Phase 4)* — when a script carries signals for more than one of `HTTP`, `CLI`, `Job`, `Consumer`. Freeform answers validate against the four-Type enum.
3. **Domain-tag assignment for a feature** *(Phase 4)* — when ≥2 plausible tags from the cover's `## Domains` list could own the feature. Default = best guess; alternatives = the other plausible tags with rationale.

**Behaviour split by invocation context:**

- **Synchronous (main session)** — prompt the user directly at the named decision points using the canonical question shape (default + 2–4 alternatives + freeform). Wait for the user before writing the affected leaves.
- **Sub-agent (Phase 4b dispatch from `atlas-bootstrap-workspace`)** — you cannot prompt. List every confirmation-worthy decision in the `=== ASSUMPTIONS ===` block of your structured return. The orchestrator runs a single batched checkpoint against the user after all sub-agents return and applies the user's resolutions before persisting your leaves.

**Question shape** (canonical):

```
[topic]: <one-line topic>
  default: <best guess>
  1) <alt-1> — <one-line rationale>
  2) <alt-2> — <one-line rationale>
  freeform: type any value to override
```

Empty submit accepts the default. Literal `accept all` accepts every default in the current batch.

**Provenance**: every confirmed decision is recorded in the affected leaf as `<topic>: <chosen value> (default | alt-N | freeform)`, either inline in the affected section or as an HTML comment immediately above it.

## See also

- Previous phase: [`atlas-identify-domains`](../atlas-identify-domains/SKILL.md)
- Multi-repo orchestrator: [`atlas-bootstrap-workspace`](../atlas-bootstrap-workspace/SKILL.md)
- Audit: [`atlas-validate`](../atlas-validate/SKILL.md)
