# repo-atlas

> A spec-driven toolkit for generating, maintaining, and querying a **Repo Atlas** — a navigable, AI-friendly knowledge graph of your codebase, sized for both single repos and multi-repo workspaces.

## What you get

- Seven coordinated skills (survey, identify-domains, generate-tree, bootstrap-workspace, refresh, validate, query) under [skills/](skills).
- Co-located templates inside each owning skill folder under `skills/<skill>/templates/`.

## Quickstart

### Single repo

```
1. Run atlas-survey-repo.
2. Run atlas-identify-domains, confirm tag list.
3. Run atlas-generate-tree.
4. Run atlas-validate.
```

Outputs: `<repo>/ATLAS.md` + `<repo>/.atlas/<feature>/feature.md` files (+ optional `_models/`, `_entrypoints/`).

### Multi-repo workspace

```
1. Run atlas-bootstrap-workspace, point at the workspace home and the in-scope repos.
2. Confirm tag lists per repo and bridge classifications.
3. Validate.
```

Outputs: `<workspace>/atlas.workspace.md` (manifest with `## Bridges` section) + per-repo atlases.

## Layouts

- **`repo-rooted`** (default) — atlas committed alongside its repo. Best for repos you own.
- **`workspace-rooted`** — atlas under `<workspace>/.atlas/<repo>/`. Best for vendored / read-only siblings.

Each skill is self-contained — schema rules, templates, and confirmation patterns are inlined in the skill bodies under [skills/](skills).

## Install as a Claude Code plugin

This repo ships a [Claude Code plugin manifest](.claude-plugin/plugin.json), so the skills can be loaded directly into Claude Code.

**Try it locally**

```bash
git clone https://github.com/SmallChX/repo-atlas.git
claude --plugin-dir ./repo-atlas
```

Inside Claude Code, the skills are exposed under the `repo-atlas:` namespace, e.g. `/repo-atlas:atlas-survey-repo`. Run `/help` to list them, and `/reload-plugins` after edits.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE).

## Status

Initial release.
