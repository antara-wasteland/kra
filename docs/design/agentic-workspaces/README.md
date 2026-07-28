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

## Scope of this copy

These notes originated as a single cross-tool wiki that was kept alongside more than one
worktree/workspace manager. That does not survive contact with real repositories: the same file
drifts in two places, and tool-specific evaluation ends up living in a repo that cannot act on it.

This copy is split along one rule:

| Content | Lives where |
|---|---|
| Shared substrate — problem framing, core insights, requirement baseline, architecture, lifecycle, storage | Duplicated by design in each substrate repo. Treat as a common charter; change it deliberately. |
| `kra`-specific — positioning, source-level gap analysis, roadmap, fork stance | **This repo only** (`kra/`, `decisions/`). |
| Sibling-tool specifics — Gion evaluation, Worktrunk/GTR feature studies, Herdr integration design | The relevant tool's own repo. Here they are reduced to one positioning page ([`neighbors.md`](neighbors.md)). |

When adding a page, ask which row it belongs to first. If it is an evaluation of a tool other
than `kra`, it does not belong here.

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
+ repo bootstrap            (cmux today, Herdr wanted)
       │                         │
       └────────────┬────────────┘
                    ▼
                   Git
```

Long-lived repository inventory/bootstrap remains a separate concern, currently handled with Mani.

## Map

Shared substrate:

- [Problem framing](01-problem-framing.md)
- [Core insights](02-core-insights.md)
- [Requirements](03-requirements.md)
- [Architecture overview](architecture/overview.md)
- [Lifecycle design](architecture/lifecycle.md)
- [Monorepo and polyrepo](architecture/monorepo-polyrepo.md)
- [Storage and performance](architecture/storage-performance.md)
- [Exploration log](exploration-log.md)

`kra` in this architecture:

- [Positioning](kra/positioning.md)
- [Gap analysis (source-level audit)](kra/gap-analysis.md)
- [Roadmap](kra/roadmap.md)
- [Proposed task workspace workflow](workflows/task-workspace.md)

Decisions:

- [Fork / upstream strategy](decisions/fork-strategy.md)
- [Open questions](decisions/open-questions.md)

Context:

- [Neighboring tools](neighbors.md)

## Design principle

Keep the bottom layers boring.

Git should own Git. The workspace manager should own workspaces. The terminal runtime should own
terminal/process sessions. Specification and agent systems should own tasks, plans, and execution policy.

The temptation to make one CLI own all of these is exactly how a useful worktree wrapper slowly
becomes a small civilization.
