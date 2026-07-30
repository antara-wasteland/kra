---
title: "Agentic Workspaces & Git Worktrees"
status: draft
---

# Agentic Workspaces & Git Worktrees

Design notes for a scalable agentic engineering workstation built around Git worktrees,
and the record of where `kra` currently sits inside that architecture.

The central insight is that **a worktree is an implementation primitive; a task workspace is the unit of work**.

A task may involve:

- one worktree from a monorepo,
- several worktrees from a polyrepo system,
- task-local plans, notes, and artifacts,
- one or more coding agents,
- dev servers, tests, and other processes,
- lifecycle setup and teardown.

The desired architecture therefore separates repository inventory, task/workspace management,
Git worktree lifecycle, terminal/process orchestration, and higher-level agent/specification workflows.

## Scope

These notes began as a cross-tool survey of worktree/workspace managers. That phase is over: `kra` is
the chosen substrate, and development happens in this fork.

So the scope here is `kra` and nothing else:

- **In scope** — why the task-first model is right, what the substrate must do, where `kra` currently
  stands against that, and what to build next.
- **Out of scope** — evaluation of other worktree or workspace managers. The comparison did its job
  and has been removed; it is recoverable from Git history if ever needed.

The one exception is the terminal runtime, which is genuinely part of `kra`'s design surface: `kra`
integrates with cmux today, and the goal is a runtime-neutral port so a different runtime can be an
adapter rather than a rewrite. That is a `kra` feature, not a tool comparison.

When adding a page, ask whether it changes what `kra` should do. If not, it does not belong here.

## Current direction

```text
OpenSpec / AXIS / plans / agent harness
                    │
                    ▼
             Task Workspace
                  (kra)
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
Git worktree lifecycle      Terminal runtime
+ repo bootstrap            (cmux today, adapter wanted)
       │                         │
       └────────────┬────────────┘
                    ▼
                   Git
```

Durable machine-wide repository inventory stays outside this picture — `kra`'s repo pool is a
per-remote object cache, not an inventory of what you own.

## Map

Why, and what the substrate must do:

- [Problem framing](01-problem-framing.md)
- [Core insights](02-core-insights.md)
- [Requirements](03-requirements.md) — the baseline the audit scores against
- [Architecture overview](architecture/overview.md)
- [Lifecycle design](architecture/lifecycle.md)
- [Monorepo and polyrepo](architecture/monorepo-polyrepo.md)
- [Storage and performance](architecture/storage-performance.md)

Where `kra` stands, and what to build:

- [Positioning](kra/positioning.md)
- [Gap analysis (source-level audit)](kra/gap-analysis.md)
- [Roadmap](kra/roadmap.md)
- [Task workspace workflow](workflows/task-workspace.md) — target flow vs today, step by step

Decisions:

- [Fork / upstream strategy](decisions/fork-strategy.md)
- [Open questions](decisions/open-questions.md)

## Design principle

Keep the bottom layers boring.

Git should own Git. The workspace manager should own workspaces. The terminal runtime should own
terminal/process sessions. Specification and agent systems should own tasks, plans, and execution policy.

The temptation to make one CLI own all of these is exactly how a useful worktree wrapper slowly
becomes a small civilization.
