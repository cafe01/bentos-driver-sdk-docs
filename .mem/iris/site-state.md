# Driver SDK Docs Site — State

## What exists

Hugo site built with the custom `driver-sdk` theme. 25 pages across 5 sections:
- **Getting Started**: 5-min quick-win, echo driver (P1). Copy-paste-run.
- **Concepts**: Three-Layer Architecture, Wire Protocol, Choosing a Pattern (the decision guide).
- **Patterns**: P1 Pure Stream, P2 Write-then-Read, P3 Event Stream, P4 Configured Stream. Each has: Linux precedent, state machine, ops contract table, poll semantics, full working example, when-to-use guidance.
- **Tutorials**: Build Your First Driver (deep kv walkthrough, P2). Build an Inference Driver (stubbed Coming Soon).
- **Reference**: Error Handling (errno mapping + per-pattern behavior), Ops Contracts (all contracts in one place).

Theme: sidebar nav with section expansion, breadcrumbs, dark code blocks, 1000px content width.

## Design decisions

- Language-agnostic text, Dart reference examples. Explicit callouts about future bindings.
- "Choosing a Pattern" organized around "how do read and write relate?" — the key insight.
- P4 deep-dive explicitly maps to inference (config=model params, write=prompt, stream=tokens).
- Getting Started vs Tutorial: GS = copy-paste quick-win (echo/P1), Tutorial = deep walkthrough (kv/P2). Zero overlap.

