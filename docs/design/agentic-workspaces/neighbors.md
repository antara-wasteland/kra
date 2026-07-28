---
title: "Neighboring Tools"
status: draft
---

# Neighboring Tools

Positioning only. Per the [scope rule](README.md#scope-of-this-copy), detailed evaluation of a tool
belongs in that tool's own repository, not here.

The category is young, and the common mistake is comparing these as if they solve the same problem.
They occupy different layers:

| Tool | Primary abstraction | Layer relative to kra |
|---|---|---|
| **kra** | ticket/task workspace | this repo — the substrate |
| Gion | declarative multi-repo workspace (plan/apply, shared bare stores) | sibling substrate; same layer, different stance |
| Worktrunk | repository worktree | below/beside — lifecycle and UX reference |
| GTR | repository worktree | below/beside — feature-completeness reference |
| gwq | global worktree inventory | navigation/discovery |
| git-wt variants | thin Git wrapper | minimal native ergonomics |
| Mani | repository inventory/bootstrap | separate concern — durable inventory |
| Herdr / cmux | terminal workspace | above — processes, panes, agent sessions |

## The distinction that matters

```text
repository-first:  repository → branches → worktrees
task-first:        task → repositories → worktrees
```

Task-first is the more general model: a monorepo task is the one-repository case. That is why the
substrate search ended at a task-first tool rather than at a worktree wrapper.

## Gion

Gion is the other task-first candidate, and the closest sibling. It is declarative and
reconciliation-oriented (plan/apply against a desired workspace state); `kra` is operational and
ticket-lifecycle-oriented. `kra`'s risk vocabulary is explicitly modelled on Gion's
(`docs/dev/spec/commands/ws/close.md` says "compute risk similar to `gion`"), and its `ws list` is
described as similar in spirit to `gion manifest ls`.

Both ideas are wanted — reproducible composition *and* convenient daily task lifecycle — but one has
to be the source of truth. That choice is recorded in
[open questions](decisions/open-questions.md#which-substrate-is-the-source-of-truth).

Gion-side analysis lives in the Gion fork. Do not add it here.

## Worktrunk and GTR

Neither is a candidate substrate; both are references.

Worth borrowing: blocking pre-hooks, background post-hooks, hook trust/approval, safe removal,
trash-before-delete, configurable paths, shell navigation, extensible custom commands. Most of these
are on the [roadmap](kra/roadmap.md).

Deliberately not borrowed: LLM-generated commit messages, model configuration, merge workflow,
PR/CI conveniences. The objection is foundational coupling, not the features themselves — a
workstation should not need an LLM API key in order to create a worktree reliably.

## Mani

A different concern that is easy to conflate: durable repository inventory and machine bootstrap,
versus ephemeral task workspaces. They coexist.

```text
Mani  = what repositories belong to this workstation
kra   = what repositories/branches belong to this task right now
```

## Herdr and cmux

Terminal runtime, above `kra`. Owns terminal workspaces, tabs/panes, running and agent processes,
and focus/navigation. It must not become the authority for Git worktree lifecycle, and it should not
separately create the worktrees `kra` already manages.

Herdr-specific integration design belongs in the Herdr work, not here; the `kra`-side prerequisite is
[roadmap 4.1](kra/roadmap.md#41-rename-the-runtime-port-upstreamable-larger).
