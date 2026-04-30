# Workspace Atlas — <Workspace Name>

> Single source of truth for the multi-repo workspace. Lists every in-scope repo and the bridges that connect them.

## Layout

<!-- One of: `repo-rooted` (default — per-repo `<repo>/ATLAS.md` + `<repo>/.atlas/`), `workspace-rooted` (`<workspace>/.atlas/<repo>/ATLAS.md`). Omit section to imply `repo-rooted`. -->

repo-rooted

## Repos

### Repo: <repo-name>

- **Role**: <one-line role>
- **Location**: <git-url-or-path>
- **Atlas**: <repo-name>/ATLAS.md  <!-- workspace-rooted: .atlas/<repo-name>/ATLAS.md -->
- **Owners**: <team-handle>
- **Domains**: <tag-1>, <tag-2>, <tag-3>
- **Summary**: <2–3 sentences>

### Repo: <repo-name>

- **Role**: <one-line role>
- **Location**: <git-url-or-path>
- **Atlas**: <repo-name>/ATLAS.md
- **Owners**: <team-handle>
- **Domains**: <tag-1>, <tag-2>
- **Summary**: <2–3 sentences>

## Cross-Repo Dependencies

<!-- Prose-edge list, alphabetical by `<from-repo>`. Each edge maps to ≥1 bullet in `## Knowledge Graph` and ≥1 sub-section in `## Bridges`. -->

- `<from-repo>` → `<to-repo>`: <one-line description of the dependency>

## Domain Map

<!-- Aggregate every domain tag across all repos. Alphabetical. A tag SHOULD belong to exactly one repo. -->

- `<tag>` → `<repo-name>`
- `<tag>` → `<repo-name>`

## Knowledge Graph

<!-- One bullet per bridge, alphabetical by bridge name. Each bullet's anchor resolves to a sub-section in `## Bridges`. -->

- [<bridge-name>](#bridge-<bridge-name>) — <kind>: <producer-repo> → <consumer-repo-1>[, <consumer-repo-2>]

## Bridges

<!-- Canonical record. One sub-section per bridge, anchored. Each ≤60 lines. See bridge-subsection.md for the exact shape. -->

### Bridge: <bridge-name> {#bridge-<bridge-name>}

- **Kind**: <api-call | shared-collection | binary-drop | submodule-include | event-stream | file-handoff>
- **Producer**: <repo>/<file-or-symbol>
- **Consumed by**:
  - <repo> — <how it consumes>
  - <repo> — <how it consumes>
- **Contract**: <one paragraph spec of payload shape, naming, semantics>
- **Discovery**: <one line — how the bridge was found>
- **Last Verified**: <producer-sha> ↔ <consumer-sha-1>, <consumer-sha-2> (<YYYY-MM-DD>)
