---
name: atlas-validate
description: Audit an existing Repo Atlas (single repo or workspace) — line budgets, required sections, alphabetical ordering, intra-repo and cross-repo reciprocity, bridge anchor integrity, and `_models/` and `_entrypoints/` thresholds. Read-only.
when_to_use: |
  Trigger phrases:
  - "is my atlas healthy?"
  - "validate atlas"
  - "check atlas budgets"

  Use this skill any time you want to confirm an atlas conforms to the schema without modifying it. Auto-activates when atlas files are open. Produces a findings report grouped by severity. Hard findings block bootstrap/refresh completion; soft findings are informational (e.g., references into not-yet-generated repos).

  Do NOT use this skill to fix issues — it's read-only. Use atlas-refresh for fixes.
allowed-tools: read_file, list_dir, file_search, grep_search, run_in_terminal
paths: [".atlas/**/*.md", "ATLAS.md", "atlas.workspace.md"]
---

# Skill: atlas-validate

> Maintenance skill. **Read-only.** Audits an atlas against the schema and produces a findings report.

## Inputs

- **Scope**: one of `repo` (a single repo's `ATLAS.md` + `.atlas/`), `workspace` (an `atlas.workspace.md` plus every repo it references), or `path` (a specific atlas file or directory for a focused audit).

## Output

```
# Atlas validation report — <scope>

## Hard findings (block completion)
- [<category>] <message> — <file>:<line>

## Soft findings (informational)
- [<category>] <message> — <file>:<line>

## Summary
- Files audited: <n>
- Hard findings: <n>
- Soft findings: <n>
```

## Audit categories

### B. Line budgets

- **Hard**: `ATLAS.md` > 80 lines.
- **Hard**: any `INDEX.md` > 200 lines.
- **Hard**: any feature leaf > 150 lines.
- **Hard**: any shared-model leaf (`_models/<m>.md`) > 80 lines.
- **Hard**: any bridge sub-section in `atlas.workspace.md` > 60 lines.
- **Hard**: repo total feature count > 50.

### C. File presence

- **Hard**: `ATLAS.md` missing.
- **Hard**: `.atlas/INDEX.md` or `.atlas/glossary.md` missing.
- **Hard**: any `<feature>/feature.md` referenced from `INDEX.md` is missing.
- **Hard**: any `_models/<m>.md` referenced from a feature's `## Models` section is missing.
- **Hard**: workspace scope: `atlas.workspace.md` missing.

### D. Forbidden directories and files

- **Hard**: any `.atlas/domains/` directory exists.
- **Hard**: any `.atlas/<feature>/{modules,entrypoints,models,bridges}/` directory exists.
- **Hard**: any per-repo `bridges/` directory exists at any depth.
- **Hard**: any `INDEX.md` exists under `.atlas/_models/` (the directory listing IS the index).
- **Hard**: any `domains/<d>/README.md` exists.

### E. Required sections

- **Hard**: cover missing one of (`## Summary`, `## Domain & Purpose`, `## Tech Stack`, `## Owners`, `## Domains`, `## Index`, `## Last Verified`) or sections out of order.
- **Hard**: cover's `## Domains` section is empty OR contains a tag not used by any feature OR omits a tag declared by a feature.
- **Hard**: feature leaf missing one of the 11 required sections (`# Feature:`, `## Breadcrumb`, `## Domain`, `## Summary`, `## Triggers`, `## Public Interface`, `## Models`, `## Code Paths`, `## External References`, `## See Also`, `## Last Verified`) or sections out of order.
- **Hard**: feature `## Domain` body is not exactly one kebab-case identifier matching a tag in the cover's `## Domains` list.
- **Hard**: feature `## Triggers` has zero bullets.
- **Hard**: feature `## Public Interface` lacks a fenced code block, OR no `<file>:<line>` line precedes the fence.
- **Hard**: feature `## Code Paths` has fewer than 5 bullets, OR any bullet lacks a `<file>:<line>` citation.
- **Hard**: feature `## See Also` body is empty (must be `_None._` or ≥1 bullet).
- **Hard**: shared-model leaf missing one of (`# Model:`, `## Breadcrumb`, `## Defined In`, `## Shape`, `## Used By`, `## Persistence`, `## Last Verified`).
- **Hard**: shared-model `## Shape` lacks a fenced code block.
- **Hard**: bridge sub-section in `atlas.workspace.md` missing one of (`Kind:`, `Producer:`, `Consumed by:`, `Contract:`, `Discovery:`, `Last Verified:`).
- **Hard**: bridge `Kind:` not one of the six enum values (`api-call`, `shared-collection`, `binary-drop`, `submodule-include`, `event-stream`, `file-handoff`).
- **Hard**: bridge `Last Verified:` missing the SHA pair format `<producer-sha> ↔ <consumer-shas>` + date.
- **Hard**: entrypoint catalog row missing the `→ [<feature>](../<feature>/feature.md)` link.

### F. Path & ordering

- **Hard**: feature path doesn't match `.atlas/<feature-kebab>/feature.md`.
- **Hard**: shared-model path doesn't match `.atlas/_models/<model-kebab>.md`.
- **Hard**: entrypoint catalog file is not exactly one of `_entrypoints/http-routes.md` or `_entrypoints/cli-commands.md`.
- **Hard**: any list-bearing index section is not alphabetically sorted.
- **Hard**: any breadcrumb segment (except the last) is not a working link.

### G. `_models/` threshold

- **Hard**: a `_models/<m>.md` leaf exists but is referenced by ≤2 features (must be inlined back into those features).
- **Hard**: an inline model entry (`- <name> — <shape>`) appears in ≥3 features (must be promoted to `_models/<name>.md` and the inlined entries replaced with links).
- **Hard**: cover lacks a `## Shared Models` section despite `_models/` being non-empty.
- **Hard**: cover has a `## Shared Models` section despite `_models/` being absent or empty.

### H. `_entrypoints/` threshold

- **Soft**: HTTP catalog absent despite the repo exposing ≥10 routes spread across ≥3 features.
- **Soft**: CLI catalog absent despite the repo exposing ≥10 CLI subcommands.
- **Hard**: a catalog row's feature link doesn't resolve.
- **Hard**: a catalog row has no corresponding entry in the named feature's `## Triggers` section.

### I. Intra-repo reciprocity

- **Hard**: a `## See Also` link from feature A to feature B exists but B's `## See Also` doesn't link back.
- **Hard**: a `## See Also` link points outside `.atlas/`.
- **Hard**: a cross-repo path appears in `## See Also`. Cross-repo links MUST live in `## External References`.

### J. Cross-repo reciprocity

- **Hard**: an `## External References` entry's repo prefix isn't listed under the workspace manifest's `## Repos`.
- **Hard**: an `## External References` bridge-anchor link doesn't resolve to a `### Bridge:` sub-section under `atlas.workspace.md`'s `## Bridges`.
- **Hard**: an `## External References` direct-feature link's path-after-the-repo-name doesn't resolve under that repo's checkout.
- **Hard**: a bridge sub-section's `Consumed by:` lists a repo that has no consumer feature carrying the bridge anchor in its `## External References`.
- **Hard**: an `## External References` entry pointing at a bridge declares a `<kind>` slot that disagrees with the bridge's `Kind:`.
- **Soft**: an `## External References` entry points at a repo that exists in `## Repos` but is `_Not yet generated._`.

### K. Knowledge-graph integrity (workspace scope only)

- **Hard**: `## Knowledge Graph` lacks an entry for a bridge sub-section that exists under `## Bridges` (orphan sub-section).
- **Hard**: `## Knowledge Graph` references a bridge whose anchor doesn't resolve to a sub-section (phantom bridge).
- **Hard**: a bridge's `Consumed by:` lists a repo that isn't in the workspace `## Repos`.
- **Soft**: a bridge has zero entries in `Consumed by:` (orphan producer — flag for human to confirm whether the bridge is still active).

### L. Metadata

- **Hard**: `## Last Verified` missing or unparseable on the cover.
- **Hard**: `## Last Verified` SHA in any leaf is in the future relative to the cover.

### M. Skill catalog (workspace scope only)

- **Hard**: a skill `SKILL.md` lacks `name`, `description`, `when_to_use`, or `allowed-tools` frontmatter.
- **Hard**: a read-only skill (no write tools in `allowed-tools`) writes a file when invoked. (Manual inspection of allowed-tools and procedure.)
- **Hard**: `len(description) + len(when_to_use) > 1536` for any skill.
- **Hard**: a `when_to_use` example trigger phrase appears in two skills' `when_to_use`.
- **Soft**: a skill has fewer than 3 example trigger phrases in `when_to_use`.

### N. Layout-mismatch (workspace scope only)

- **Hard**: workspace manifest declares `## Layout: workspace-rooted` AND any `<repo>/ATLAS.md` or `<repo>/.atlas/` exists for a repo listed under `## Repos`.
- **Hard**: workspace manifest declares (or implies) `## Layout: repo-rooted` AND `<workspace>/.atlas/<repo>/` exists for any atlassed repo.
- **Soft**: `## Layout` body is anything other than `repo-rooted` or `workspace-rooted` (frozen enum).

### O. Layout-form (cross-repo links)

- **Hard**: `## Layout` is `workspace-rooted` AND any cross-repo link in any atlas file uses the `<repo>/.atlas/...` form.
- **Hard**: `## Layout` is `repo-rooted` (or absent) AND any cross-repo link in any atlas file uses the `.atlas/<repo>/...` form.
- Applies to `## External References` direct-feature links and the workspace manifest's per-repo `Atlas:` field. Bridge-anchor links (which use a relative path to `atlas.workspace.md`) are layout-agnostic.

### P. Workspace-index (workspace scope only)

- **Hard**: `## Layout: workspace-rooted` AND `<workspace>/.atlas/INDEX.md` is missing.
- **Hard**: `## Layout: workspace-rooted` AND `<workspace>/.atlas/INDEX.md`'s `## Repos` set diverges from the manifest's atlassed `### Repo:` entries.
- **Hard**: `## Layout` is `repo-rooted` (or absent) AND `<workspace>/.atlas/INDEX.md` exists.

## Procedure

1. **Detect scope**. Workspace manifest entry → workspace. `ATLAS.md` entry → repo. Otherwise → path.
2. **Read `## Layout`** from the workspace manifest if present (default `repo-rooted` when absent or no manifest exists). Layout-aware categories N–P use this value.
3. **Walk the tree**. For repo scope: read `ATLAS.md`, then every file under `.atlas/` (resolved per the active layout). For workspace scope: read the manifest, then walk every repo it references at the layout-mandated location, plus the manifest's `## Bridges` section. For path scope: walk only the supplied path.
4. **Apply audit categories B–P** in order. For each violation, emit a finding with category, message, and file:line.
5. **Aggregate** into the report shape above. Group by severity. Sort within each group alphabetically by `<file>:<line>`.
6. **Return** the report. Do not modify any file.

## Self-application

This skill is the validator that `atlas-bootstrap-workspace`, `atlas-generate-tree`, and `atlas-refresh` hand off to. Their hand-off contract: the calling skill is "complete" only when this skill reports zero hard findings.

## Layout fork

Layout-aware. The active layout is determined by reading `## Layout` from `atlas.workspace.md` (default `repo-rooted` when no manifest exists or no section is set). The layout governs:

- Where atlas files are read from (forked under category C path resolution).
- Which form cross-repo direct-feature links must take (category O).
- Whether `<workspace>/.atlas/INDEX.md` is required vs. forbidden (category P).

This skill never writes; the only output is the findings report.

## See also

- Refresh: [`atlas-refresh`](../atlas-refresh/SKILL.md)
- Bootstrap: [`atlas-bootstrap-workspace`](../atlas-bootstrap-workspace/SKILL.md)
