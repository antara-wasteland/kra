---
title: "Open Questions"
status: draft
---

# Open Questions

Adoption gates for the workspace substrate. The [gap analysis](../kra/gap-analysis.md) resolved
several of these against the actual `kra` implementation; those are recorded under
[Resolved](#resolved) with what settled them.

## Still open

### Which substrate is the source of truth

Declarative desired workspace state (Gion) or operational ticket workspace lifecycle (`kra`)?

A hybrid may emerge, but one has to be authoritative. The audit does not settle this — it only shows
that `kra`'s operational side is already built and specified, and that what it lacks is additive.
The symmetric audit of the sibling substrate belongs in that repo; this question is answered once
both exist.

Until then, this fork proceeds on the `kra` side without foreclosing the other.

### Hook semantics

The design surface is sketched in [roadmap 1.2](../kra/roadmap.md#12-lifecycle-hooks-upstreamable--the-headline-gap),
but these are unsettled:

- exact event set (is `pre-attach` per repository worth its complexity?),
- retry semantics on partial failure,
- workspace-level versus repository-level hook precedence and ordering,
- whether hook failure blocks `ws open`, or only marks the workspace not-ready,
- how failure state is represented for archived and reopened workspaces.

Nothing here is blocked on new information — it is a specification to write.

### Hook trust model

First-execution approval and re-approval on content change are settled in principle. Unsettled:
what the trust unit is (file hash, directory hash, repository), whether trust is per-root or per
`KRA_HOME`, and how non-interactive automation acquires trust without an interactive session
(a `kra hook trust` command, or refuse-only).

### Monorepo focus

Should a workspace record a `focus` path inside a repository, for navigation, scoped bootstrap,
agent cwd, and status presentation?

Leaning yes — it is one optional field in `repos_restore[]`
([roadmap 2.3](../kra/roadmap.md#23-repo-focus-upstreamable)). The constraint is firm: it must not
create extra Git worktrees. Open only on whether hooks should receive it as scope, or merely as
context.

### Runtime recovery

How should terminal runtime state recover when the runtime restarts, when task metadata survives but
the terminal workspace disappears, or when a terminal workspace exists for an archived task?

`kra`'s storage answer is right — runtime mapping is non-canonical and commands degrade past it —
and `doctor` already detects stale locks and broken links. What is missing is a runtime-side
reconciliation pass. Best specified together with the runtime port
([roadmap 4.1](../kra/roadmap.md#41-rename-the-runtime-port-upstreamable-larger)) rather than against
cmux specifically.

### Path layout flexibility

`workspaces/<id>/repos/<alias>` is structural in `kra`, not templated. Fixed layout is what makes
filesystem-first root detection and deterministic agent paths work. Is templated layout actually
wanted, or is that a requirement inherited from repository-first tools that had no root of their own?

Leaning: leave it fixed. Record the reason before anyone "fixes" it.

## Resolved

### Archive semantics — resolved

*Question: when a task closes, should its worktrees remain attached, be detached while preserving
branches, be converted to lighter metadata, or depend on policy?*

Detach, preserve branches, keep task artifacts. `ws close` removes worktrees while archiving
`notes/` and `artifacts/` to `archive/<id>/`, and `.kra.meta.json`'s `repos_restore[]` carries
`repo_uid`, `remote_url`, `alias`, `branch`, and `base_ref` — enough for `ws reopen` to refetch and
re-attach. Lighter metadata *is* the archived form.

One nuance to remember: `repos_restore[]` records the branch, not a commit, so reopen restores the
branch as it stands then.

### Shared repository pool — resolved

*Question: should the task manager own its own bare cache, or reuse an existing durable layout?*

`kra` owns one, at `$KRA_HOME/repo-pool/<host>/<owner>/<repo>.git`, shared across all roots and
workspaces on the machine — so there is one object store per remote, not per task and not per root.
`repo gc` reclaims it behind gates that include references from *other* known roots. No duplicate
object stores; nothing further to decide.

### State storage — resolved

*Question: should a small `.state/` directory record operational metadata without becoming a second
source of truth for Git facts?*

Yes, and `kra` already draws the line correctly
(`docs/dev/spec/concepts/state-store.md`): `.kra.meta.json` is canonical workspace state; runtime
files under `.kra/state/` are explicitly non-canonical, must degrade gracefully when missing or
stale, and read-only commands must not "repair" canonical metadata from them. Git facts are
recomputed from Git — risk classification is live at every `ws close`, `ws purge`, and
`ws dashboard`.

The remaining work is to *populate* that store with hook approvals, bootstrap completion, allocated
ports, and process handles — a roadmap item, not an open question.
