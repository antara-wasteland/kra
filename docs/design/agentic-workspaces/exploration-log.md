---
title: "Exploration Log"
status: historical
---

# Exploration Log

This document captures the evolution of the reasoning rather than only the final architecture.

It is a historical record — read it for *why* the architecture landed where it did, not for current
state. Where it names decisions as open, check [open questions](decisions/open-questions.md) first;
several were resolved by auditing the code. Its closing item — compare candidate substrates at
source level — is answered for `kra` in the [gap analysis](kra/gap-analysis.md).

## 1. Starting from worktree wrappers

The exploration began with `satococoa/wtp` and the observation that Git worktree managers are appearing rapidly as parallel coding agents make worktrees much more important.

The initial question was whether there were stable, mature equivalents.

The early landscape included:

- CodeRabbit Git Worktree Runner (GTR),
- Worktrunk,
- gwq,
- several `git-wt` implementations,
- newer agent-oriented managers.

The first conclusion was that the category itself is young. Native Git remains the mature substrate.

## 2. GTR looked pragmatic, but Bash raised concern

GTR had an unusually complete operational surface:

- hooks,
- setup/copy,
- cleanup,
- diagnostics,
- agent/editor launching,
- shell integration,
- trust gating.

It looked like a strong default.

But an earlier custom worktree manager had already exposed a weakness of shell-based orchestration: when many repositories or parallel operations are involved, serial Bash flows become slow and awkward.

A later Go rewrite naturally supported parallel repository operations.

That experience made a compiled implementation more attractive for the long-term substrate.

## 3. Worktrunk looked architecturally polished

Worktrunk stood out for lifecycle quality:

- pre/post hooks,
- safe cleanup,
- background removal,
- navigation,
- path templates,
- extensibility,
- strong worktree ergonomics.

But its scope also expanded into:

- LLM-generated commit messages,
- model configuration,
- merge workflow,
- PR/CI conveniences.

Those features are not necessarily bad. The concern is foundational coupling.

The workstation should not require an LLM API key merely because it needs reliable worktree creation.

## 4. The forgotten earlier thread: Gion

The conversation returned to an earlier exploration of Gion.

Gion had been attractive because it was not merely a repository-scoped worktree wrapper.

Its model supported task workspaces composed from multiple repository worktrees, backed by shared bare Git storage.

That matched an earlier polyrepo workflow where one task could require frontend and backend repositories together.

The key distinction became:

```text
Worktrunk:
repository → branches → worktrees

Gion:
task → repositories → worktrees
```

## 5. Monorepos clarified the abstraction

A concern emerged: what happens with monorepos?

The answer sharpened the model.

A monorepo task still needs only one Git worktree because Git worktrees operate at repository granularity.

```text
TASK-A/
└── monorepo/
```

A polyrepo task may contain several:

```text
TASK-B/
├── frontend/
├── backend/
└── infra/
```

Therefore, task-first composition works for both.

The task abstraction is more general than the repository abstraction.

## 6. Mandatory requirement: post-create lifecycle

The non-negotiable feature became explicit:

> Newly created worktrees must be bootstrappable automatically.

Examples:

```text
uv sync
pnpm install
direnv allow
submodule initialization
generated config
local service setup
```

This made Worktrunk's lifecycle model particularly valuable even if Worktrunk itself was not adopted.

The conclusion was to borrow lifecycle primitives rather than workflow policy.

## 7. What to borrow from Worktrunk

The most valuable transferable ideas were:

1. blocking pre-hooks,
2. background post-hooks,
3. hook trust/approval,
4. safe removal,
5. trash/background deletion,
6. configurable paths,
7. shell navigation,
8. extensible custom commands.

Things not to copy into the substrate:

- LLM commit generation,
- merge policy,
- PR policy,
- task decomposition.

## 8. Mani was separated from the problem

Mani had been used for a different concern: repository bootstrapping and durable workspace inventory.

It should not be confused with task/worktree orchestration.

The split became:

```text
Mani
= durable repository inventory

Gion/kra-like manager
= ephemeral task workspace
```

## 9. kra added an operational task model

`tasuku43/kra` revealed another useful layer.

Where Gion feels declarative and reconciliation-oriented, kra feels ticket/workspace lifecycle-oriented.

It introduces an explicit task/ticket workspace that can contain:

- repository worktrees,
- notes,
- artifacts,
- archive/reopen lifecycle.

The task key itself became an especially appealing stable identity.

## 10. cmux versus Herdr

kra integrates with cmux, but the workstation uses Herdr.

Rather than adopting cmux solely for kra, the better design is a terminal/runtime adapter abstraction.

```text
task workspace manager
        │
        ├── cmux adapter
        ├── Herdr adapter
        └── directory/no-op adapter
```

A Herdr PR therefore makes sense if implemented generically rather than as scattered special cases.

## 11. Ownership boundary with Herdr

Herdr should manage:

- terminal workspaces,
- panes,
- tabs,
- agents,
- commands,
- process status.

The task manager should manage:

- task identity,
- worktree attachment,
- archive/reopen,
- workspace metadata.

Herdr should not separately create/manage the same Git worktrees when paired with the task manager.

## 12. Current architectural direction

The emerging target is:

```text
Specifications / plans / agent harness
                 │
                 ▼
        task workspace manager
          (kra/Gion-like)
           │             │
           ▼             ▼
    repo lifecycle     Herdr
           │             │
           └──────┬──────┘
                  ▼
                 Git
```

Repository-owned scripts form the stable lifecycle contract.

```text
scripts/worktree-bootstrap
scripts/worktree-teardown
scripts/verify
```

## 13. Current decision status

No final tool has been irrevocably selected.

The strongest architectural direction is:

- task-first,
- native Git underneath,
- Go/compiled implementation for parallel polyrepo work,
- Worktrunk-quality lifecycle,
- kra/Gion-quality task composition,
- Herdr for terminal/process orchestration,
- higher-level agent/spec systems kept separate.

The next practical investigation should compare Gion and kra at source level around:

- hook extensibility,
- parallel execution,
- task state model,
- archive/reopen,
- cleanup,
- machine-readable APIs,
- runtime integration boundaries.

That comparison should decide whether to upstream changes, fork one of them, or build a very small integration layer around them.
