---
title: "Workspace Substrate Requirements"
status: draft
---

# Requirements

This is the requirement baseline for the workspace substrate, written before any tool was chosen.

It is deliberately tool-neutral. [`kra/gap-analysis.md`](kra/gap-analysis.md) scores the current
`kra` implementation against every item on this page; [`kra/roadmap.md`](kra/roadmap.md) sequences
the gaps.

## Mandatory

### Post-create bootstrap

A newly created worktree should be able to run deterministic setup such as:

```bash
uv sync
pnpm install --frozen-lockfile
git submodule update --init --recursive
direnv allow
./scripts/worktree-bootstrap
```

The hook should receive useful context such as:

```text
TASK_ID
WORKSPACE_PATH
REPO_NAME
WORKTREE_PATH
BRANCH
INDEX
```

### Ergonomic CLI

Core operations should be cheap enough to become muscle memory:

```text
create
open / enter
list
status
exec
remove
cleanup
```

Interactive fuzzy navigation is useful, but every important operation should also be scriptable.

### Safe cleanup

Cleanup should distinguish:

- clean and integrated,
- dirty,
- unpushed,
- diverged,
- unknown,
- stale,
- archived.

Destructive operations should fail safely by default.

### Monorepo and polyrepo support

One task should be able to represent:

```text
TASK-A/
└── platform/          # monorepo
```

or:

```text
TASK-B/
├── frontend/
├── backend/
└── infra/             # polyrepo
```

without changing the overall workflow model.

### Efficient storage and execution

Avoid full clones per task.

Use shared Git object stores through worktrees/bare repositories and shared package caches where safe.

### Machine interface

Prefer:

- stable exit codes,
- JSON output,
- non-interactive flags,
- deterministic paths,
- explicit state.

Agents should not need to scrape pretty terminal output.

## Strongly desired

- hook trust/approval,
- blocking and background hooks,
- archive/reopen,
- trash-before-delete,
- doctor/repair,
- path templates,
- custom subcommands,
- shell navigation protocol,
- parallel operations across repositories.

## Explicitly not required at this layer

- LLM-generated commit messages,
- model API keys,
- task decomposition,
- autonomous merging,
- PR policy,
- CI decision policy,
- specification ownership.

Those belong above the workspace substrate.
