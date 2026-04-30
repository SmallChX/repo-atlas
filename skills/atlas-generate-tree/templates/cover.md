# <Repo Name>

> One-screen home for `<repo-name>`. ≤80 lines. Cover-only — chapter detail lives in `.atlas/`.

## Summary

<2–4 sentences: what this repo does in plain English. Avoid implementation jargon.>

## Domain & Purpose

<One paragraph: which business or platform domain(s) this repo owns; what is in scope and what is explicitly out of scope.>

## Tech Stack

- Language(s): <Go 1.22 / TypeScript 5.x / Python 3.12 / …>
- Framework(s): <Gin / NestJS / FastAPI / …>
- Datastores: <Postgres / Redis / S3 / …>
- Build / runtime: <Bazel / npm / Cargo / Docker / Kubernetes>

## Owners

- <team-handle> — <area-of-ownership>
- <team-handle> — <area-of-ownership>

## Domains

> Frozen, kebab-case tag list. 3–10 tags. Each tag is the value used by every feature's `## Domain` line.

- <tag-1>
- <tag-2>
- <tag-3>

## Shared Models

<!-- Include this section ONLY when `_models/` is non-empty. Omit otherwise. -->

- [<model-name>](.atlas/_models/<model-name>.md) — <one-line role>

## Bridges Produced

<!-- Optional. Producer repos MAY list bridges they own with anchor links to atlas.workspace.md. Omit for very small repos. -->

- [<bridge-name>](../atlas.workspace.md#bridge-<bridge-name>) — <kind>: <one-line role>

## Index

- [Feature index](.atlas/INDEX.md)
- [Glossary](.atlas/glossary.md)
- [Dependencies](.atlas/dependencies.md) <!-- delete this bullet if no dependencies.md -->
- [HTTP routes catalog](.atlas/_entrypoints/http-routes.md) <!-- delete unless threshold met -->
- [CLI commands catalog](.atlas/_entrypoints/cli-commands.md) <!-- delete unless threshold met -->

## Last Verified

<commit-sha-short> (<YYYY-MM-DD>)
