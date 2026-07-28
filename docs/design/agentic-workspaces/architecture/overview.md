---
title: "Architecture Overview"
status: draft
---

# Architecture Overview

## Layers

```text
┌─────────────────────────────────────────────┐
│ Specifications / Plans / Agent Harness      │
│ OpenSpec, AXIS, AGENTS.md, execution policy │
└─────────────────────┬───────────────────────┘
                      │ task identity
                      ▼
┌─────────────────────────────────────────────┐
│ Task Workspace Manager                      │
│ kra                                         │
│                                             │
│ task key, repo composition, archive/reopen, │
│ worktree attachment, workspace metadata     │
└──────────────┬──────────────────┬───────────┘
               │                  │
               ▼                  ▼
┌──────────────────────┐  ┌───────────────────┐
│ Repo lifecycle       │  │ Terminal runtime  │
│ hooks/bootstrap      │  │ cmux / Herdr      │
│ status/cleanup       │  │ panes/processes   │
└──────────┬───────────┘  └─────────┬─────────┘
           │                        │
           └────────────┬───────────┘
                        ▼
                       Git
```

## Ownership boundaries

### Task workspace manager owns

- task/workspace identity,
- workspace root,
- attached repositories,
- desired worktrees,
- workspace notes/artifacts,
- archive/reopen/purge lifecycle,
- persistent mapping to terminal runtime IDs.

### Git owns

- branch state,
- commits,
- refs,
- merge semantics,
- remote tracking,
- object database.

### Repository scripts own

- dependency bootstrap,
- generated files,
- local service initialization,
- verification,
- teardown.

### Terminal runtime owns

(cmux today, Herdr wanted — see [gap analysis](../kra/gap-analysis.md#8-runtime-adapter-abstraction--partial))

- terminal workspaces,
- tabs/panes,
- running processes,
- agent processes,
- focus/navigation within terminal UI.

### Higher orchestration owns

- what task to perform,
- which agent gets it,
- completion criteria,
- specification state,
- commit/PR policy.

## Why this boundary matters

A lower-level tool can be reused by humans, Codex, Claude, Pi, CI, or future agents.

Once the workspace layer starts requiring model API keys or deciding how to merge code, it becomes much harder to replace or compose.
