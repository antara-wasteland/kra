---
title: "kra in this Architecture"
status: draft
---

# kra in this Architecture

Upstream: `tasuku43/kra`. This repository is a fork — see [fork strategy](../decisions/fork-strategy.md).

## Why kra was chosen as the substrate

`kra` makes the ticket/workspace itself explicit:

```text
TASK-1234/
├── workspace.md
├── notes/
├── artifacts/
├── .kra.meta.json
└── repos/
    ├── backend/
    └── frontend/
```

with a lifecycle of create → open → close (archive) → reopen → purge.

The explicit task key is the reason. Task identity survives:

- terminal recreation,
- agent restarts,
- branch changes,
- archive and reopen,
- multi-repository work.

That is precisely the [core insight](../02-core-insights.md) that the task, not the repository, is
the unit of work — expressed as a filesystem layout rather than as a convention an agent has to
remember.

## What kra owns

Per the [architecture overview](../architecture/overview.md), `kra` is the task workspace manager
layer, and only that:

- task/workspace identity and root layout,
- attached repositories and their worktrees,
- workspace notes and artifacts,
- archive / reopen / purge lifecycle,
- canonical workspace metadata (`.kra.meta.json`),
- persistent mapping to terminal runtime IDs.

It does not own merge semantics, PR policy, task decomposition, or agent execution policy — and it
should not acquire them.

## What the audit found

The full evidence is in the [gap analysis](gap-analysis.md). In one paragraph:

`kra` is unusually strong on state discipline — filesystem-first canonical metadata, live risk
classification (`clean`/`dirty`/`unpushed`/`diverged`/`unknown`), recoverable multi-phase close via
lifecycle journals, a shared bare repo pool with cross-root-aware GC, a consistent JSON envelope with
stable exit codes and dry-run preflight on destructive commands, and `doctor --fix --plan|--apply`.
It is absent on extensibility: there are no lifecycle hooks at all, therefore no hook trust, no
background process supervision, and no way for a repository to declare its own setup. Those gaps are
additive rather than architectural, which is why the fork does not need to fight the design to close
them.

## Relationship to the terminal runtime

`kra` integrates with cmux, but terminal integration is not the fundamental workspace model — the
[state store spec](../../../dev/spec/concepts/state-store.md) already treats runtime mapping as
non-canonical data that commands must degrade past.

The remaining coupling is vocabulary, not architecture: cmux appears in ~27 source files and in
state filenames, template scaffold, and doctor checks. Abstracting that behind a runtime-neutral port
is [roadmap 4.1](roadmap.md#41-rename-the-runtime-port-upstreamable-larger); Herdr becomes an adapter
after it, not before.

## Target shape

`kra` as a substrate owns:

```text
task ID
workspace metadata
repo attachments
archive / reopen
notes / artifacts
runtime mapping
lifecycle hook execution   ← the missing piece
```

while delegating repository setup to hooks the repository itself owns, and terminal/process behavior
to a runtime adapter.
