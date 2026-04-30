<!--
Bridge sub-section template. Lives ONLY under `## Bridges` in `atlas.workspace.md`. ≤60 lines per sub-section. Anchor `#bridge-<name>` is mandatory.
-->

### Bridge: <bridge-name> {#bridge-<bridge-name>}

<!-- provenance: <topic>: <chosen value> (default | alt-N | freeform)   ← only when interactive-confirmation resolved a decision -->

- **Kind**: <api-call | shared-collection | binary-drop | submodule-include | event-stream | file-handoff>
- **Producer**: <producer-repo>/<file-or-symbol>
- **Consumed by**:
  - <consumer-repo> — <how it consumes>
  - <consumer-repo> — <how it consumes>
- **Contract**: <one paragraph spec of payload shape, naming, semantics>
- **Discovery**: <one line — how the bridge was found>
- **Last Verified**: <producer-sha> ↔ <consumer-sha-1>, <consumer-sha-2> (<YYYY-MM-DD>)
