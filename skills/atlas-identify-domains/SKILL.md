---
name: atlas-identify-domains
description: Propose 3–10 domain tags for a repo before atlas generation — kebab names, rationales, and per-tag feature-count estimates. Use after atlas-survey-repo and before atlas-generate-tree; produces a domain-tag proposal, no files written.
when_to_use: |
  Trigger phrases:
  - "split this repo into domains"
  - "DDD this repo"
  - "propose domain tags for this repo"

  Use this skill when a survey note exists and the user wants a concrete domain-tag decomposition (3–10 tags, repo total ≤50 features) before any atlas files are written. Output is a proposal the user reviews.

  Do NOT use this skill if the user has already named the tags they want — go straight to atlas-generate-tree. Domain tags are NOT directories; each feature carries its tag in its `## Domain` section.
allowed-tools: read_file, list_dir, file_search, grep_search, semantic_search
paths: [".atlas/**/*.md", "ATLAS.md", "atlas.workspace.md"]
---

# Skill: atlas-identify-domains

> Phase 2 of 4 in the generation family. Read-only. Produces a domain-tag proposal that the user confirms before `atlas-generate-tree` writes the tree.

## Inputs

- **Survey note** from `atlas-survey-repo` (or equivalent reconnaissance the user supplies).
- **Repo path**.

## Output (returned in conversation, no files written)

```
# Domain-tag proposal: <repo-name>

## Tags (3–10)

### <tag-kebab>
- Rationale: <why this is its own bounded context, grounded in survey evidence>
- Source roots: <dir-1>/, <dir-2>/
- Estimated features: ~<n>      (single integer; no per-chapter breakdown — chapters do not exist)
- Candidate features: <feature-1>, <feature-2>, …

### <next-tag-kebab>
…

## Repo-wide totals
- Total estimated features: <sum across tags>      (HARD CAP: 50)
- Shared-model candidates: <names> — referenced by ≥3 features each (will be promoted to `_models/`)
- Entrypoint catalog flag: HTTP=<yes|no, ≥10 routes / ≥3 features>; CLI=<yes|no, ≥10 commands>

## Notes
- <ambiguities, judgement calls, or things the user should confirm>
- <provenance lines from confirmation: `<topic>: <chosen value> (default | alt-N | freeform)`>
```

## Procedure

### 1. Re-read the survey

Take `## Candidate domain tags` and `## Candidate features` as starting points. Don't blindly accept either — verify against source code.

### 2. Apply heuristics, in order

| Signal                                | Strength | What it tells you                                          |
|---------------------------------------|----------|------------------------------------------------------------|
| Top-level source directory name       | Strong   | `src/billing/` is almost always its own tag.               |
| Package / namespace cluster           | Strong   | Repeated prefix across files (`com.example.billing.*`).    |
| Data ownership (writers of a table)   | Medium   | Cross-cutting writers may indicate a missing service seam. |
| CODEOWNERS section                    | Medium   | Team boundary, not always a domain boundary.               |
| Test file organisation                | Weak     | Confirms but rarely defines a tag.                         |
| Route prefix                          | Weak     | `/api/billing` may map to billing — verify the handler.    |

When signals conflict, prefer the strongest. Reject any tag candidate that has only weak evidence.

### 3. Estimate per-tag feature count

For each surviving tag, count user-meaningful capabilities visible in source — entry points × user verbs. Aim order-of-magnitude correct.

**Do NOT estimate modules / entrypoints / models / bridges separately.** Modules don't exist. Entrypoints fold into features. Models inline unless ≥3 features share them. Bridges live in the workspace manifest, not the per-repo atlas.

### 4. Apply the 50-feature cap

> A repo SHALL NOT exceed 50 features in total. Hard limit.

If the sum across tags exceeds 50, choose between:

- **Repo split**: this repo is doing too much; recommend splitting into two atlassed repos (rare; usually requires a code refactor first).
- **Feature split**: an umbrella feature is too coarse — propose splitting one or two features into 2–3 narrower features each. Surface the candidates in `## Notes` so the user can confirm during `atlas-generate-tree`'s feature-naming decision point.

The split rationale must be a real seam in the code, not arbitrary alphabetisation.

### 5. Apply the 3–10 tag ceiling

If you end with **fewer than 3** tags: under-segmenting. Look for sub-areas inside the largest tag with their own dependency profile.

If you end with **more than 10** tags: over-segmenting. Two tags that share most of their dependency profile should merge.

### 6. Detect shared-model candidates

For each model-like entity (ORM entity, schema file, DTO, wire type) referenced by candidate features, count how many features touch it. Models touched by **≥3 features** become `_models/<name>.md` leaves; flag them in the `## Repo-wide totals` block. Models touched by ≤2 features stay inlined and need no flag.

### 7. Detect entrypoint-catalog flag

Use the survey's `## Entry-point inventory`:

- HTTP catalog flag = `yes` iff the repo exposes ≥10 routes spread across ≥3 candidate features.
- CLI catalog flag = `yes` iff the repo exposes ≥10 CLI subcommands.

Below threshold → flag = `no`. The catalog file is **not** written; triggers live only in their owning features.

### 8. Write the proposal

Emit the proposal block. Each tag gets its own `### <tag-kebab>` subsection with rationale, source roots, feature estimate, and candidate-feature list. Add the repo-wide totals block.

### 9. Surface notes

`## Notes` covers:

- Tags that almost made the cut but were merged or dropped.
- Estimates with high uncertainty (a directory with mixed concerns).
- Cross-cutting features whose tag-assignment is genuinely ambiguous (≥2 plausible tags) — flag for `atlas-generate-tree` to confirm at write time.
- Provenance lines from any confirmation surfaced in this skill.

## Interactive Confirmation

The proposal step is itself a confirmation; this section formalises it and adds an extra decision point.

**Decision points** (single batched checkpoint at end of skill, before returning the proposal):

1. **Domain-tag list** — the proposed list of 3–10 tags. Default = the AI's best decomposition; alternatives = at least one alternate split (merge two candidates into an umbrella, or split the largest further) with one-line rationale.
2. **Domain-boundary line within a single source directory** — whenever a proposed boundary would assign different files in the same source directory to different tags. Default = the proposed split; alternative = assigning the whole directory to one tag, with rationale.

If neither the overall proposal nor any boundary is genuinely ambiguous, surface only an acknowledgement-style confirmation of the proposal. Do NOT manufacture alternatives.

**Question shape** (canonical):

```
[topic]: <one-line topic, e.g., "split src/integrations/ between integrations and platform">
  default: <best guess>
  1) <alt-1> — <one-line rationale>
  2) <alt-2> — <one-line rationale>
  freeform: type any value to override
```

Empty submit accepts the default. Literal `accept all` accepts every default in the batch.

**Provenance**: every confirmed decision is recorded in the proposal's `## Notes` section as a single line `<topic>: <chosen value> (default | alt-N | freeform)` so `atlas-generate-tree` carries the provenance into the leaves it writes.

## Hand-off

Return the proposal in the conversation. The user reviews and either confirms or edits before invoking `atlas-generate-tree`. Do not write any files.

## Layout fork

Layout-agnostic. Domain-tag proposal depends only on the source repo's structure, never on where the atlas will be written. Mention is included for catalog completeness; no behaviour changes per layout.

## See also

- Previous phase: [`atlas-survey-repo`](../atlas-survey-repo/SKILL.md)
- Next phase: [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md)
