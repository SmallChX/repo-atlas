---
name: atlas-refresh
description: Incrementally update an existing atlas to reflect code changes since its last verification, including bridge changes (in atlas.workspace.md), `_models/` promotion/demotion, and cross-repo reciprocity.
when_to_use: |
  Trigger phrases:
  - "my atlas is stale"
  - "refresh atlas"
  - "update atlas after these changes"

  Use this skill when an atlas exists for a repo and code has landed since its last verification. Touches only the leaves the diff requires; preserves untouched leaves' Last Verified to signal partial freshness. Re-evaluates `_models/` promotion/demotion thresholds on every run.

  Do NOT use this skill to bootstrap from scratch — use atlas-generate-tree (single repo) or atlas-bootstrap-workspace (multi-repo). Do NOT use to validate — use atlas-validate.
allowed-tools: read_file, list_dir, file_search, grep_search, semantic_search, run_in_terminal, create_file, replace_string_in_file, multi_replace_string_in_file
---

# Skill: atlas-refresh

> Maintenance skill. **Writes files.** Diff-driven update of an existing atlas.

## Inputs

- **Atlas path**: filesystem path to the repo containing `ATLAS.md`.
- **Target commit** *(optional)*: SHA the atlas should be reconciled to. Defaults to current `HEAD`.

## Output

A modified atlas tree. Every touched file's `## Last Verified` is updated. Diff scope is the minimum needed to reflect the changes — this is not a regeneration.

## 1. Staleness check

Read `## Last Verified` on `ATLAS.md`. Decide refresh scope:

1. `<previous-sha>` missing or unparseable → escalate to full `atlas-generate-tree` run.
2. `<previous-sha>` is **not an ancestor** of the target commit (history rewrites, force-pushes) → escalate to full regeneration.
3. `<previous-sha> == <target-sha>` → nothing to do.
4. Otherwise → proceed to step 2.

## 2. Compute the diff

`git diff --name-status <previous>..<target>`. Classify each changed path:

| Source change                                  | Atlas impact                                                                                                |
|------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| File added inside a known feature              | Possibly extends `## Code Paths`; may add a new feature if the file is the first under a new capability    |
| File deleted                                   | Referencing features must drop the bullets; if last file of a feature, remove the feature                  |
| Signature/exports changed                      | Update referencing features' `## Public Interface` excerpt and `## Code Paths` line numbers                |
| Body only, no signature change                 | Likely no atlas impact                                                                                      |
| Route added/removed                            | Update affected feature's `## Triggers`; re-evaluate `_entrypoints/http-routes.md` threshold                |
| CLI command added/removed                      | Update affected feature's `## Triggers`; re-evaluate `_entrypoints/cli-commands.md` threshold               |
| Migration / schema added                       | Update `## Models` entries in referencing features                                                          |
| Package manifest changed                       | Re-evaluate `dependencies.md` (when present) external section                                               |
| New top-level source directory                 | Possibly a new domain tag — re-run `atlas-identify-domains` on this repo                                    |
| **Producer-side bridge contract changed**      | Update `atlas.workspace.md`'s bridge sub-section `Contract:` and `Last Verified:` SHA pair (orchestrator)  |
| **Consumer added/removed for a bridge**        | Update bridge sub-section's `Consumed by:`; touch consumer's `## External References` (orchestrator)        |
| **New cross-repo edge introduced**             | Add new bridge sub-section in `atlas.workspace.md` + reciprocal back-link in consumer feature (orchestrator)|

Build a **working set** — the list of features and shared models to add, modify, or delete. Touching anything outside this set is forbidden in a refresh run.

## 3. Apply leaf updates

For each entity in the working set:

- **Add feature**: create `<feature>/feature.md` from [./templates/feature.md](../atlas-generate-tree/templates/feature.md) (yes, refresh borrows generate-tree's templates), populate from new source. Leave `## See Also` and `## External References` empty for now.
- **Modify feature**: read the existing leaf, edit only the sections affected by the change. Preserve the breadcrumb and `## Domain` line. Preserve reciprocal sections unless the change invalidates them.
- **Delete feature**: remove the directory. Record the path so step 5 can purge inbound back-links.
- **Add/modify/delete shared-model**: same pattern under `_models/`.

## 4. Re-evaluate `_models/` thresholds

After step 3, count cross-feature model references:

- **Inline model now referenced by ≥3 features** → promote: create `_models/<m>.md`, replace inlined entries in those features with links, add the model to the cover's `## Shared Models` section.
- **Existing `_models/<m>.md` now referenced by ≤2 features** → demote: inline the model back into its referencing features, delete `_models/<m>.md`, remove from cover's `## Shared Models`. If the section becomes empty, delete the section entirely.

This is a hard rule, not a judgement call. The threshold is enforced on every refresh.

## 5. Regenerate `INDEX.md` and `_entrypoints/` catalogs

Whenever a feature is added, removed, or renamed:

- Regenerate `.atlas/INDEX.md`'s `## Features by Domain` section. Under each `### Domain: <tag>` heading, one bullet per feature whose `## Domain` matches, alphabetical, summary harvested from feature's first `## Summary` sentence.

Whenever the route/command count crosses a catalog threshold:

- Now ≥10 routes / ≥3 features and `_entrypoints/http-routes.md` absent → write the catalog from [./templates/http-routes.md](../atlas-generate-tree/templates/http-routes.md).
- Now <10 routes (or <3 features) and the catalog exists → delete the catalog.
- Same logic for CLI.

If only a feature's body changed without touching its `## Summary` first sentence, no `INDEX.md` regeneration is needed.

## 6. Reciprocal-link maintenance

For every feature added or modified:

1. Inspect each `## See Also` (intra-repo) link. Add missing back-link to the target. Remove dangling outbound links.
2. Inspect each `## External References` (cross-repo) entry. Verify the target file exists in the named repo (for direct-feature links) OR the manifest anchor resolves (for bridge links). Verify reciprocity: direct-feature targets carry a back-link; bridges' `Consumed by:` lists this repo.
3. Sort both sections alphabetically.

For every feature deleted:

1. Grep the workspace for inbound `## See Also` and `## External References` references and remove them.
2. For deletions of features that consumed bridges, the orchestrator updates the bridge sub-section's `Consumed by:` — flag those edits for the next workspace-scope refresh if running at single-repo scope.

After the pass, both reciprocal invariants hold across the full atlas (and across the workspace, when run at workspace scope).

## 7. Feature-budget overflow trigger

After step 5, check the repo total:

- > 50 features → pause the refresh, propose a feature-naming split (one or two umbrella features into 2–3 narrower ones) using the `atlas-generate-tree` confirmation pattern, and ask the user to confirm before performing the rename.
- After confirmation: create new feature directories, move files, regenerate `INDEX.md`, and update every cross-feature `## See Also` link to the new path.

A refresh run that triggers a split SHOULD be its own commit, separate from substantive content changes.

## 8. Update `## Last Verified`

- Set `ATLAS.md` `## Last Verified` to the target SHA + today's date.
- For each feature whose body was touched, set its `## Last Verified` to the same SHA + date.
- Features whose bodies were not touched keep their previous `## Last Verified` — that's the signal for partial freshness.
- For shared-model leaves whose body changed, update `## Last Verified`.
- For bridge sub-sections whose contract or consumer set changed (workspace-scope refresh only), update the SHA pair `<producer-sha> ↔ <consumer-sha>[, ...]` plus today's date.

## 9. Hand off to atlas-validate

Run [`atlas-validate`](../atlas-validate/SKILL.md) and resolve every hard finding before declaring the refresh complete. Apply the redaction checklist from [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md#redaction-checklist) — refreshed content can introduce new secrets.

## Layout fork

Before reading any atlas file, read `## Layout` from the workspace manifest (default `repo-rooted` when absent or no manifest exists). Fork the source paths exactly as in [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md#layout-fork):

- `repo-rooted`: `<repo>/ATLAS.md` + `<repo>/.atlas/`.
- `workspace-rooted`: `<workspace>/.atlas/<repo>/ATLAS.md` + `<workspace>/.atlas/<repo>/`.

Cross-repo links written or rewritten during steps 5 and 6 MUST use the form mandated by the active layout.

## See also

- Validation: [`atlas-validate`](../atlas-validate/SKILL.md)
- Initial generation: [`atlas-generate-tree`](../atlas-generate-tree/SKILL.md)
- Multi-repo orchestrator: [`atlas-bootstrap-workspace`](../atlas-bootstrap-workspace/SKILL.md)
