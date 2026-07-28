---
title: "Problem Framing"
status: draft
---

# Problem Framing

## Why worktrees became important

Parallel coding agents changed the practical value of Git worktrees.

A human often works sequentially:

```text
branch A → branch B → branch C
```

Parallel agents instead want:

```text
agent A → branch/worktree A
agent B → branch/worktree B
agent C → branch/worktree C
```

Each agent needs an isolated filesystem while retaining cheap access to the same repository history.

Git worktrees already provide the correct primitive: multiple working directories sharing one Git object database.

The remaining problem is operational ergonomics.

## The missing operational layer

Native `git worktree` does not by itself solve:

- ergonomic creation and navigation,
- project bootstrap after creation,
- environment/config propagation,
- cleanup and stale worktree management,
- safe branch deletion,
- status across many worktrees,
- task-level grouping,
- polyrepo tasks,
- terminal/session association,
- interrupted-agent recovery.

This created a wave of newer worktree managers, many explicitly designed around parallel coding agents.

## The trap

There are two very different products hiding under the phrase "agent worktree manager."

### Worktree lifecycle manager

```text
create
bootstrap
enter
inspect
execute
remove
cleanup
```

### Agent workflow orchestrator

```text
decompose task
launch model
generate commit
merge
open PR
watch CI
decide completion
```

The first is infrastructure.

The second is policy.

The workstation should avoid accidentally coupling the infrastructure layer to a particular model provider, agent CLI, merge strategy, or task system.
