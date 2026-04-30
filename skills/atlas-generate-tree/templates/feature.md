# Feature: <Feature Name>

> Single leaf for one capability. ≤150 lines. 11 required sections in this exact order.

## Breadcrumb

[<repo-name>](../../ATLAS.md) → [Features](../INDEX.md) → <Feature Name>

## Domain

<exactly one kebab-case tag from the cover's `## Domains` list>

## Summary

<2–4 sentences: what this feature does in plain English. First sentence becomes the INDEX bullet.>

## Triggers

<!-- ≥1 bullet. How this feature is invoked: HTTP route, CLI command, scheduled job, queue consumer, library entrypoint. Each bullet cites a `<file>:<line>`. -->

- <kind>: <signature> — <file>:<line>

## Public Interface

<!-- Code excerpt of the canonical entrypoint signature. ≤30 lines. Precede the fence with the `<file>:<line>` line. -->

`<file>:<line>`

```<lang>
<excerpted signature>
```

## Models

<!-- Inline when used by ≤2 features. Promote to `_models/<m>.md` and link out when used by ≥3 features. -->

- <ModelName> — { <field>: <type>, … }
- [<SharedModelName>](../_models/<shared-model-name>.md) — <one-line role>

## Code Paths

<!-- ≥5 bullets. Each bullet is `<file>:<line>` followed by a one-line note about role. -->

- <file>:<line> — <role: registration | dispatch | core | validation | persistence | …>
- <file>:<line> — <role>
- <file>:<line> — <role>
- <file>:<line> — <role>
- <file>:<line> — <role>

## External References

<!-- Cross-repo links only. Bridge anchors first, direct-feature links second. Omit section body and write `_None._` if there are no cross-repo references. Forbidden in `## See Also`. -->

- [<bridge-name>](../../../atlas.workspace.md#bridge-<bridge-name>) — <kind>: <one-line purpose>
- [<other-repo>/<other-feature>](../../../<other-repo>/.atlas/<other-feature>/feature.md) — <one-line purpose>

## See Also

<!-- Intra-repo only. Reciprocal — every link MUST have a back-link in the target. Body `_None._` when empty. -->

- [<sibling-feature>](../<sibling-feature>/feature.md) — <one-line relation>

## Last Verified

<commit-sha-short> (<YYYY-MM-DD>)
