---
name: atlas-bootstrap-workspace
description: Bootstrap a multi-repo workspace — survey every in-scope repo, generate per-repo atlases in two phases, write the workspace manifest with all bridges as anchored sub-sections, and validate workspace-wide. The workspace manifest is the single source of truth for cross-repo bridges.
when_to_use: |
  Trigger phrases:
  - "atlas this monorepo"
  - "bootstrap workspace"
  - "build the knowledge graph for these repos"

  Use this skill when initialising the atlas across two or more related repositories. This skill orchestrates the survey/identify/generate skills for each repo, discovers cross-repo bridges, and writes the workspace manifest including its `## Bridges` section (anchored sub-sections — the canonical record of every bridge contract).

  Do NOT use for a single repo — use atlas-survey-repo + atlas-identify-domains + atlas-generate-tree directly. Do NOT create per-repo `bridges/` directories — bridges live ONLY in `atlas.workspace.md`.
allowed-tools: all
---

# Skill: atlas-bootstrap-workspace

> Phase 4 of 4 in the generation family. **Writes files across many repos.** Orchestrates the per-repo skills and is the **sole writer** of the workspace manifest including its `## Bridges` section.

## Inputs

- **Workspace home**: directory where `atlas.workspace.md` will live.
- **Repo list**: paths or URLs to each in-scope repo, plus a one-line role for each.
- **Owners** *(optional)*: per-repo owner handles.

## Output

```
<workspace-home>/atlas.workspace.md         # workspace manifest with `## Bridges` section (anchored sub-sections)
<each-repo>/ATLAS.md + .atlas/              # per-repo atlas (cover + INDEX + glossary + features + optional _models/_entrypoints)
```

There SHALL be no `bridges/` directory in any per-repo atlas. Every bridge lives as one anchored `### Bridge: <name> {#bridge-<name>}` sub-section under `## Bridges` in `atlas.workspace.md`.

## Templates (co-located with this skill)

- [./templates/atlas.workspace.md](./templates/atlas.workspace.md) — workspace manifest shape.
- [./templates/bridge-subsection.md](./templates/bridge-subsection.md) — the seven-field anchored sub-section.
- [./templates/workspace-rooted-INDEX.md](./templates/workspace-rooted-INDEX.md) — `<workspace>/.atlas/INDEX.md` shape (workspace-rooted layout only).

## Six-step procedure

> Step 4 (per-repo atlas generation) is split into **4a** (synchronous skeleton phase, dependency-leaf order) and **4b** (parallel sub-agent dispatch for feature population). Step 5 (bridge persistence) runs after 4b — it is **manifest-only**.

### Step 1 — Survey every in-scope repo

For each repo, run [`atlas-survey-repo`](../atlas-survey-repo/SKILL.md). Aggregate the results. Do **not** start writing atlases yet — the bridge graph is computed across surveys before any tree is materialised.

### Step 2 — Discover bridges across the workspace

Walk the aggregated surveys to detect cross-repo edges and classify each into one of the six frozen kinds:

| Kind                | Discovery signal                                                                            |
|---------------------|--------------------------------------------------------------------------------------------|
| `api-call`          | OpenAPI / HTTP / gRPC client code in repo X referencing an endpoint defined in repo Y      |
| `shared-collection` | The same DB collection / table / blob namespace appears in 2+ repos                        |
| `binary-drop`       | A vendor / install script in repo X copies a prebuilt artefact from a sibling repo Y       |
| `submodule-include` | `.gitmodules` in repo X points at a workspace sibling Y                                    |
| `event-stream`      | The same topic / queue name appears in 2+ repos (one publishes, others subscribe)          |
| `file-handoff`      | The write path of a producer process in repo X matches the read path in repo Y            |

For each detected edge, decide:

1. **Producer repo** — the repo whose code defines the contract.
2. **Consumer repos** — every repo that reads, calls, or imports the producer's surface.
3. **Bridge name** (kebab, ≤40 chars) — derived from the contract's most identifying token (topic name, collection name, gRPC method name, library name). The name becomes the anchor `#bridge-<name>` in `atlas.workspace.md`.

If no clear producer exists (multiple writers of the same shared resource), record the edge in `## Knowledge Graph` AND under `## Bridges` with `Producer:` listing all writers; surface the ambiguity in the post-survey confirmation batch (see [Interactive Confirmation](#interactive-confirmation)).

### Step 3 — Compose the workspace manifest

Create `atlas.workspace.md` at the workspace home from [./templates/atlas.workspace.md](./templates/atlas.workspace.md). Required sections in order: optional `## Layout`, `## Repos`, `## Cross-Repo Dependencies`, `## Domain Map`, `## Knowledge Graph`, `## Bridges`.

#### `## Repos`

One `### Repo: <name>` entry per in-scope repo, alphabetical. Capture: name, role, location (git URL or filesystem path), atlas path (relative to workspace home, exists after step 4), owners, domains (filled after step 4 — the union of each repo's `## Domains` tag list), summary.

#### `## Cross-Repo Dependencies`

Prose-edge list, alphabetical by `<from-repo>`. Each edge maps to ≥1 entry in `## Knowledge Graph` and ≥1 sub-section in `## Bridges`.

#### `## Domain Map`

Aggregate every domain tag across all repos. One bullet per tag: `- <tag> → <repo>`, alphabetical. A tag SHOULD belong to exactly one repo. Conflicts are flagged for human resolution before declaring the manifest complete.

#### `## Knowledge Graph`

One bullet per discovered bridge, alphabetical by bridge name; each bullet's link resolves to its anchored sub-section in `## Bridges`:

```
- [<bridge-name>](#bridge-<bridge-name>) — <kind>: <producer-repo> → <consumer-repo-1>[, <consumer-repo-2>, ...]
```

#### `## Bridges`

The canonical record of every bridge contract. Empty for now; populated by Step 5. Sub-section shape from [./templates/bridge-subsection.md](./templates/bridge-subsection.md):

```markdown
### Bridge: <name> {#bridge-<name>}

<!-- provenance: ... -->                  (only when interactive-confirmation resolved a decision)

- **Kind**: <one of the six frozen kinds>
- **Producer**: <repo>/<file-or-symbol>
- **Consumed by**:
  - <repo> — <how it consumes>
  - <repo> — <how it consumes>
- **Contract**: <one paragraph spec of payload shape, naming, semantics>
- **Discovery**: <one line — how the bridge was found>
- **Last Verified**: <producer-sha> ↔ <consumer-sha-1>, <consumer-sha-2>, …  (YYYY-MM-DD)
```

Each sub-section ≤60 lines.

### Step 4 — Generate per-repo atlases in two phases

#### 4a. Skeletons in dependency-leaf order (synchronous)

Order the repos so producers come **first**, then consumers (so consumer features can reference producer-side anchors that already resolve in the manifest). For each repo, in order:

1. Run [`atlas-identify-domains`](../atlas-identify-domains/SKILL.md) on its survey and confirm the tag list with the user.
2. Materialise the **skeleton only** at the per-layout target: cover `ATLAS.md` (`## Domains` tag list filled), `.atlas/INDEX.md` (`## Features by Domain` with body `_None._` per `### Domain: <tag>` heading for now), `.atlas/glossary.md`, optionally `.atlas/dependencies.md`. There SHALL be no feature directories yet, no `_models/`, no `_entrypoints/` — those land in Phase 4b.

Skeletons are intentionally tiny — the cover + INDEX + glossary is the entire skeleton for any repo, regardless of domain count.

If a repo is **out of scope** (the user excluded it but other in-scope repos depend on it), proceed without atlassing it. Cross-repo references pointing into it become `_Not yet generated._` placeholders.

#### 4b. Populate features in parallel via sub-agents

Once every in-scope repo has a skeleton, dispatch one sub-agent **per repo** in parallel to run [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md). Each sub-agent's working set is its own repo subtree — no shared state, safe to parallelise.

Per-sub-agent prompt template (one invocation per repo):

```
Run atlas-generate-tree for repo <repo-name> at <repo-root>.
Atlas root: <atlas-root>      (workspace-rooted: <workspace>/.atlas/<repo>/  | repo-rooted: <repo>/.atlas/)
Layout: <repo-rooted | workspace-rooted>
Domain tags (already declared in skeleton): <tag-1>, <tag-2>, ...
Workspace context (read-only):
  - sibling repos: <list>
  - bridge anchors this repo participates in (already in atlas.workspace.md#bridges):
    - <bridge-name> (kind, role producer|consumer)
    - ...

Thoroughness: THOROUGH. You MUST open and read source files — directory listings + filenames are not sufficient. Plan to open at least:
  - 2–3 source files per feature (registration site + dispatch site + at least one core implementation site).
  - 1 source file per shared-model candidate (to excerpt the canonical Shape).

Depth bar — every feature leaf MUST satisfy these minimums or the run fails:
  - All 11 sections present in order, ≤150 lines.
  - `## Domain` is exactly one kebab tag from the cover's `## Domains` list.
  - `## Summary` 2–4 sentences.
  - `## Triggers` ≥1 cited bullet.
  - `## Public Interface` fenced code block excerpted from source with `<file>:<line>` above the fence.
  - `## Code Paths` ≥5 `<file>:<line>` bullets each with a one-line note.
  - Models inlined unless ref-count ≥3 across features (Phase 5 promotion).
  - Every cross-link resolves.

Constraints:
  - Touch ONLY <repo>'s feature directories: `.atlas/<feature>/`, `.atlas/_models/`, `.atlas/_entrypoints/`.
  - Do NOT modify ATLAS.md, INDEX.md, glossary.md, dependencies.md — those are owned by the orchestrator.
  - Do NOT touch atlas.workspace.md or any sibling repo.
  - Do NOT create any directory matching `domains/`, `modules/`, `bridges/`, or per-feature `entrypoints/`.
  - Respect schema budgets: feature ≤150 lines, shared-model ≤80 lines, INDEX ≤200 lines, ≤50 features per repo.
  - Citations MUST be honest: every `<file>:<line>` MUST resolve in the current working tree. Do NOT guess line numbers.

Return:
  - List of features written (with paths).
  - Whether `_models/` was populated (and which models — those with ref-count ≥3).
  - Whether `_entrypoints/` was populated (and which catalogs).
  - Candidate cross-repo back-links you observed (consumer-side `## External References` pairs the orchestrator should validate in Step 5).
  - Any unresolved questions.

=== ASSUMPTIONS ===
Mandatory section. List every confirmation-worthy decision you made (one with ≥2 plausible alternatives) using the canonical question shape. Do NOT prompt the user — you are dispatched in a parallel context. The orchestrator will batch these for the user after all sub-agents return.

Format (one entry per decision, max 20 entries):

  - decision: <one-line topic, e.g., "feature naming for bootstrap-build-system">
    chose: <your best guess; this becomes the default>
    alternatives:
      - value: <alt-1>
        rationale: <one line>
      - value: <alt-2>
        rationale: <one line>
    rationale: <one line — why you picked `chose`>

If no confirmation-worthy decisions arose, write the section header followed by `(none)`.
```

Use the agent-dispatch tool of the host environment (e.g., `runSubagent` / a parallel agent pool). Wait for all sub-agents to complete before proceeding.

After they all return:

- **Run the post-dispatch batched confirmation checkpoint** ([Interactive Confirmation](#interactive-confirmation)). Collect every `=== ASSUMPTIONS ===` block, deduplicate identical decisions across repos, present the union as a single batched question set (≤10 per batch; themed sub-batches on overflow). The user MAY submit empty / `accept all` to take every default at once. Apply the user's resolutions to the leaves before they are persisted.
- Re-aggregate each per-repo `INDEX.md`'s `## Features by Domain` section: under each `### Domain: <tag>` heading, one bullet per feature whose `## Domain` matches.

#### 4c. Constraints on parallelism

- Each sub-agent owns exactly one repo's feature directories (`.atlas/<feature>/`, plus optional `_models/` and `_entrypoints/`). No two sub-agents may write to the same path.
- Sub-agents are **forbidden** to edit `atlas.workspace.md` or any sibling repo. Bridges and consumer-side cross-repo back-links are orchestrator-owned (Step 5) to keep cross-repo invariants under a single writer.
- If a sub-agent attempts to create any `domains/`, `modules/`, `bridges/`, or per-feature `entrypoints/` directory, reject the return and retry that repo.
- If a sub-agent fails or times out, retry that repo only; the others' results stand. After exhausting retries, mark that repo's `INDEX.md` with `_None._` placeholders and surface a hard finding without rolling back successful repos.
- Hosts without a parallel-dispatch primitive run Phase 4b sequentially per repo; output is byte-equivalent.

### Step 5 — Persist bridges in the workspace manifest

For each bridge in `## Knowledge Graph`:

1. **Manifest sub-section**: ensure a `### Bridge: <name> {#bridge-<name>}` sub-section exists under `## Bridges` of `atlas.workspace.md`. Required fields per [./templates/bridge-subsection.md](./templates/bridge-subsection.md): `Kind`, `Producer`, `Consumed by` (bullet list), `Contract`, `Discovery`, `Last Verified`. Optional `<!-- provenance: ... -->` HTML comment when interactive-confirmation resolved a decision (bridge kind, producer-vs-consumer, bridge name).
2. **Last Verified pair**: `<producer-sha>` ↔ `<consumer-sha-1>, <consumer-sha-2>, ...` plus today's date.
3. **Consumer-side back-links**: for each consumer repo, open the relevant feature's `## External References` and add an entry `- [<bridge-name>](<rel>/atlas.workspace.md#bridge-<bridge-name>) — <kind>: <one-line purpose>`. Use the candidate references each sub-agent flagged in its return summary. The orchestrator is the sole writer of these back-links.
4. **Optional `## Bridges Produced` on producer cover**: each producer repo MAY (recommended, not required) add a `## Bridges Produced` section to its `ATLAS.md` listing bridge names with anchor links to the manifest. Keeps the cover compact; opt-out for very small repos.

There SHALL be no per-repo `bridges/` directory at any phase. Any consumer feature whose `## External References` cannot be precisely placed (because the relevant feature isn't yet identified) leaves the entry pending; `atlas-validate` will surface it as a soft finding.

### Step 6 — Workspace-wide validation gate

Run [`atlas-validate`](../atlas-validate/SKILL.md) at workspace scope. Resolve every hard finding before declaring the bootstrap complete. Specifically:

- Every `## External References` link resolves to a real file or to a manifest anchor that exists.
- Every bridge's `Consumed by` list has a matching consumer-side `## External References` entry (or a soft-finding pending placeholder).
- Bridge `Kind` values match the kind declared in consumer-side `## External References` entries.
- Workspace manifest `## Knowledge Graph` enumerates every bridge sub-section in `## Bridges` (no orphans either way).

## Interactive Confirmation

Surface a confirmation question only when the AI identifies **≥2 plausible alternatives**.

**Decision points** (each surfaced as a batched checkpoint; ≤10 per batch, themed sub-batches on overflow):

1. **Bridge kind** *(Step 2)* — every detected cross-repo edge whose heuristic signals are consistent with more than one of the six frozen kinds. Freeform answers validate against the enum.
2. **Bridge name** *(Step 2)* — when the contract carries no obviously-identifying token (multiple plausible kebab names, ≤40 chars).
3. **Producer-vs-consumer** *(Step 2)* — when both repos write to the shared resource and either could plausibly own the bridge.
4. **Post-Phase-4b assumption batch** *(after Step 4b)* — the union of every sub-agent's `=== ASSUMPTIONS ===` block, deduplicated. Run BEFORE the orchestrator persists sub-agent leaves; user resolutions are applied at persist time.

**Question shape** (canonical):

```
[topic]: <one-line topic>
  default: <best guess>
  1) <alt-1> — <one-line rationale>
  2) <alt-2> — <one-line rationale>
  freeform: type any value to override
```

Empty submit accepts the default. Literal `accept all` accepts every default in the current batch. Any freeform value targeting a frozen enum (e.g., bridge kind) MUST validate against the enum or be re-prompted.

**Provenance**: every confirmed decision is recorded as a single line `<topic>: <chosen value> (default | alt-N | freeform)` in the artefact it produces — inline in the affected section, or as an HTML comment immediately above it.

## Hand-off

Tell the user:

1. Where `atlas.workspace.md` lives.
2. Which repos got an atlas this run (and which were out of scope but referenced).
3. The discovered bridges, grouped by kind, with anchor links to the manifest.
4. Any flagged ambiguities (duplicate domains, multi-writer edges with no clear producer, missing consumer-side back-links).
5. The cadence for refreshing — typically: refresh on every notable structural PR, or weekly.

## Choosing the manifest's home

| Option                                  | Use when                                                            |
|-----------------------------------------|---------------------------------------------------------------------|
| Dedicated `*-workspace` repo            | 3+ repos, multiple teams, or workspace tooling is anticipated.      |
| Inside the most-central repo            | 2–3 repos with one clear hub (e.g., the API gateway).               |

Document the choice in the manifest's preamble so future contributors know where to maintain it.

## Layout fork

This skill accepts an optional `layout` argument (`repo-rooted` | `workspace-rooted`). Defaults:

- If an `atlas.workspace.md` already exists at the workspace home, the value of its `## Layout` section (or `repo-rooted` when absent) is the default.
- If no manifest exists, the default is `repo-rooted`.

If the caller passes a `layout` argument that disagrees with the existing manifest's `## Layout`, **refuse** to proceed and emit an error directing the user to choose the layout matching the existing manifest. Bootstrap is not a migration tool; no migration tool is provided.

Per-layout write targets:

| Layout              | Per-repo cover                                  | Per-repo subtree                              | Workspace index                            |
|---------------------|-------------------------------------------------|-----------------------------------------------|--------------------------------------------|
| `repo-rooted`       | `<repo>/ATLAS.md`                               | `<repo>/.atlas/`                              | _none_                                     |
| `workspace-rooted`  | `<workspace>/.atlas/<repo>/ATLAS.md`            | `<workspace>/.atlas/<repo>/`                  | `<workspace>/.atlas/INDEX.md` (REQUIRED)   |

In `workspace-rooted` mode, after Step 5 generate `<workspace>/.atlas/INDEX.md` from [./templates/workspace-rooted-INDEX.md](./templates/workspace-rooted-INDEX.md), populating `## Repos` from the manifest's atlassed entries.

Bridges live exclusively in `atlas.workspace.md`'s `## Bridges` section under both layouts. Anchor links from consumer-side features use a relative path to the manifest plus the `#bridge-<name>` anchor.

Write `## Layout` into `atlas.workspace.md` (before `## Repos`) for `workspace-rooted`; for `repo-rooted` you MAY omit the section (implicit default).

## See also

- Per-repo skills: [`atlas-survey-repo`](../atlas-survey-repo/SKILL.md), [`atlas-identify-domains`](../atlas-identify-domains/SKILL.md), [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md)
- Validation: [`atlas-validate`](../atlas-validate/SKILL.md)
- Refresh later: [`atlas-refresh`](../atlas-refresh/SKILL.md)
- Bridges: [docs/atlas-schema/bridges.md](../../docs/atlas-schema/bridges.md)
- Cross-repo references: [docs/atlas-schema/cross-repo-references.md](../../docs/atlas-schema/cross-repo-references.md)
- Interactive confirmation pattern: [docs/atlas-schema/interactive-confirmation.md](../../docs/atlas-schema/interactive-confirmation.md)
- Skill catalog: [docs/atlas-schema/skill-catalog.md](../../docs/atlas-schema/skill-catalog.md)
