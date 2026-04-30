---
name: atlas-survey-repo
description: Read-only reconnaissance of a repository before atlas generation — capture HEAD SHA, manifests, top-level layout, candidate domain tags, and entry-point types. Phase 1 of atlas generation; emits a structured survey note, writes no files.
when_to_use: |
  Trigger phrases:
  - "survey this repo"
  - "what's in this repo"
  - "before I atlas this, what does this repo look like"

  Use this skill when the user wants a read-only reconnaissance pass before deciding whether (or how) to atlas a repository. Output is a structured survey note in the conversation that downstream skills (atlas-identify-domains, atlas-generate-tree) consume.

  Do NOT use this skill if the user has already provided a domain-tag list — go straight to atlas-generate-tree. Do NOT enumerate "modules" — modules are not a separate chapter.
allowed-tools: read_file, list_dir, file_search, grep_search, semantic_search, run_in_terminal
paths: [".atlas/**/*.md", "ATLAS.md", "atlas.workspace.md"]
---

# Skill: atlas-survey-repo

> Phase 1 of 4 in the generation family. Read-only. Emits a survey note that the user (or `atlas-identify-domains`) reviews before any atlas files are written.

## Inputs

- **Repo path**: filesystem path to the repository root.
- **Repo name** *(optional)*: defaults to directory basename.

## Output (returned in conversation, no files written)

```
# Survey: <repo-name>

## Identity
- HEAD SHA: <full 40-char SHA, or "uncommitted-<short-sha>" if dirty>
- Branch: <current branch>
- Remote: <origin URL, or "(no remote)">
- Worktree clean: yes | no

## Manifests
- <package.json | pyproject.toml | go.mod | ...>: <terse fact (language, framework, runtime)>

## Top-level layout
- <dir-1>/ — <inferred role>
- <dir-2>/ — <inferred role>

## Author intent (READMEs, ARCHITECTURE.md)
<one paragraph synthesised from the repo's own docs, or "(no top-level docs)">

## CI / deploy
- <workflow file>: <what it builds/deploys>

## Owners
- @<team-or-handle> (from CODEOWNERS, or "(no CODEOWNERS)")

## Candidate features (preliminary, for atlas-identify-domains)
- <feature-1> — <one-line user-meaningful capability + suspected entry-point/trigger>
- <feature-2> — <…>

## Entry-point inventory
- HTTP: <count> route(s); spread across ~<n> features
- CLI:  <count> command(s)
- Job:  <count> scheduled job(s)
- Build: <count> build/install entrypoint(s) detected

## Candidate domain tags (preliminary)
- <kebab-tag-1> — <one-line rationale grounded in source roots / package clusters / CODEOWNERS>
- <kebab-tag-2> — <…>

## Open questions
- <thing that's ambiguous and the user should resolve before atlas-identify-domains>
```

## Procedure

### 1. Capture identity

```
git -C <repo> rev-parse HEAD
git -C <repo> rev-parse --abbrev-ref HEAD
git -C <repo> remote get-url origin
git -C <repo> status --porcelain
```

If `git rev-parse HEAD` fails, set `HEAD SHA: (no git)` and continue (the user is working off an unversioned tree). If `status --porcelain` is non-empty, set `Worktree clean: no` and use `uncommitted-<short-sha>` as the SHA (downstream `## Last Verified` fields propagate this verbatim).

### 2. Read manifests

Open any of: `package.json`, `pyproject.toml`, `setup.py`, `requirements*.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `Gemfile`, `composer.json`, `*.csproj`, `mix.exs`, `pubspec.yaml`, `CMakeLists.txt`. Capture language, framework hints, notable scripts.

### 3. List top-level layout

`list_dir` the repo root. Infer a one-line role per directory from naming conventions (`src/`, `app/` → source; `tests/`, `spec/` → tests; `cmd/`, `bin/` → entry points; `migrations/`, `db/` → schema; `scripts/`, `tools/` → tooling; `docs/` → docs).

### 4. Read author intent

Read up to 200 lines each of `README.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`. Synthesise one paragraph; do not invent details.

### 5. Read CI / deploy configs

Walk `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`, `.circleci/config.yml`. One bullet per workflow with what it builds and (if discernible) where it deploys.

### 6. Read CODEOWNERS

Read `CODEOWNERS` (or `.github/CODEOWNERS`, `docs/CODEOWNERS`). Extract team handles. Note path-scoped sections — they hint at domain-tag boundaries.

### 7. Inventory entry points (NOT separate leaves — counts only)

Scan for entry-point signals so downstream skills can decide whether the optional `_entrypoints/` catalog threshold (≥10 routes / ≥10 CLI commands) is met:

| Type   | Source signals                                                                                |
|--------|-----------------------------------------------------------------------------------------------|
| HTTP   | route definitions (`@Get`, `app.post`, OpenAPI, Express/Nest/Flask/Gin/Echo)                 |
| CLI    | `bin/`, `cmd/`, `argparse`, `commander`, `cobra`, `clap`                                      |
| Job    | cron registrations, `@Cron`, `bull`, `sidekiq`, `celery`, `CronJob`, scheduled-task manifests |
| Build  | top-level build / install scripts (`build.sh`, vendor scripts, `install.sh`)                  |

Report **counts only** in the survey, not per-trigger leaves. The owning feature (identified later) carries each trigger in its `## Triggers` section. The catalog files are reverse-indexes that exist only above threshold.

### 8. Propose candidate features

A feature is a **user-meaningful capability** — what someone says in a sentence ("Issue refund", "Patch a Windows host", "Run vulnerability scan"). It usually spans multiple files and surfaces through one or more triggers identified in step 7.

Aim for an order-of-magnitude estimate; the precise count is `atlas-identify-domains`' output. Keep totals under the 50-feature-per-repo cap.

### 9. Propose candidate domain tags

Apply heuristics, in order:

1. Top-level source-directory names — `src/billing/`, `src/identity/` are strong tag signals.
2. Package / namespace clusters — files sharing a prefix.
3. CODEOWNERS path scopes.
4. Data ownership clusters — modules that write the same table or topic.

Aim for **3–10 tags**. Each candidate carries a one-line rationale grounded in steps 1–6 evidence.

### 10. Surface open questions

Anything ambiguous — two top-level dirs that could be the same domain, missing CODEOWNERS, an unfamiliar framework, a script whose Type is genuinely uncertain — appears under `## Open questions`. The user resolves these before invoking `atlas-identify-domains`.

## Interactive Confirmation

Surface a confirmation question only when the AI identifies **≥2 plausible alternatives**.

**Decision points** (single batched checkpoint at end of survey, before returning the note; ≤10 per batch):

1. **Tech-stack inference** — when manifests indicate more than one stack with comparable weight (e.g., a Python `pyproject.toml` plus a substantial `package.json`).
2. **Entry-point Type assignment** — for any script that could plausibly be classified as more than one of `HTTP`, `CLI`, `Job`, or `Build`. Freeform answers validate against the enum.

If neither ambiguity exists, do not prompt; return the survey directly.

**Question shape** (canonical):

```
[topic]: <one-line topic>
  default: <best guess>
  1) <alt-1> — <one-line rationale>
  2) <alt-2> — <one-line rationale>
  freeform: type any value to override
```

Empty submit accepts the default. Literal `accept all` accepts every default in the batch.

**Provenance**: every confirmed decision is recorded in the survey note as a single line of the form `<topic>: <chosen value> (default | alt-N | freeform)` so `atlas-identify-domains` and `atlas-generate-tree` can carry the provenance into the leaves they write.

## Hand-off

Return the survey note in the conversation. Do not write any files. Do not propose modules — modules are not a separate chapter; their content folds into features.

## Layout fork

Layout-agnostic. Surveying source code is independent of where the atlas will be written. The survey output SHOULD note the active workspace layout (read `## Layout` from `<workspace>/atlas.workspace.md` if present; default `repo-rooted`) so downstream skills can plan write targets without re-reading the manifest.

This skill never writes files; its read-only profile is unaffected by the layout fork.

## See also

- Next phase: [`atlas-identify-domains`](../atlas-identify-domains/SKILL.md)
