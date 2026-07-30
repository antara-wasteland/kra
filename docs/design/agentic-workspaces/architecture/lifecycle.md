---
title: "Lifecycle Design"
status: draft
---

# Lifecycle Design

The target lifecycle model for `kra`. The useful part is the lifecycle abstraction itself — it must
stay orthogonal to merge workflow, so that adopting it does not commit `kra` to any opinion about how
code gets integrated.

## Proposed minimal lifecycle

```text
create
  ├── pre-create      blocking
  ├── create worktree
  ├── post-create     blocking setup
  └── post-create-bg  background processes

enter
  ├── pre-enter
  ├── navigation/runtime focus
  └── post-enter

archive
  ├── pre-archive
  ├── stop/detach runtime
  ├── preserve workspace
  └── post-archive

remove
  ├── safety analysis
  ├── pre-remove
  ├── detach worktrees
  ├── move disposable data to trash
  └── post-remove

gc
  └── delete eligible trash/stale state
```

## Blocking versus background hooks

Blocking hooks establish invariants.

Examples:

```text
uv sync
pnpm install
generate config
database migration
```

A failed blocking hook means the workspace is not fully ready.

Background hooks start disposable runtime activity.

Examples:

```text
dev server
file watcher
local worker
```

These should be tracked and cancellable rather than making creation wait forever.

## Hook trust

Checked-in hook definitions are executable code.

Project-level hooks should require explicit approval before first execution and again when their executable definition changes.

Local user configuration can have a different trust policy.

## Idempotency

Bootstrap should be safe to retry.

A workspace should be able to represent:

```text
created
bootstrap-failed
ready
archived
remove-pending
```

rather than pretending creation is atomic when five external commands are involved.

## Trash before delete

Removal can first move disposable workspace data to a trash location and detach Git state safely.

Permanent deletion can happen asynchronously or during GC.

This improves perceived performance and creates an undo/recovery window.
