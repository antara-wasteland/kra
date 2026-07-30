---
title: "Fork and Upstream Strategy"
status: draft
---

# Fork and Upstream Strategy

This repository is a fork of `tasuku43/kra`, and **this fork is where the work happens**. Upstreaming
is an option to keep open per change, not a gate a change has to pass before it can be built.

This page records how to keep that option cheap.

## Current state: zero divergence

`rishwanth-th/kra` `main` is identical to `tasuku43/kra` `main` at `1fdcd0b` — no commits ahead, none
behind. There is no fork-local behavior to maintain yet and nothing to reconcile.

That is the cheapest possible starting position. The point of the rules below is not to defer work to
upstream; it is to make sure that when a change *is* worth offering upstream, it can be offered
without a rewrite — and, more selfishly, that the fork can still take upstream's future commits
without a merge fight.

## The fork is not a disagreement

A fork usually signals that upstream does not want something architectural:

- task-first workspaces,
- polyrepo composition,
- lifecycle extensibility,
- runtime abstraction,
- machine-first operation.

The [gap analysis](../kra/gap-analysis.md) found that upstream `kra` already holds four of those five
positions. Only lifecycle extensibility is absent, and upstream's own backlog shows it absent as an
unfilled gap rather than a rejected proposal: all 17 backlog epic files are fully checked off, and
none of them ever raised hooks. Nothing in `kra`'s stated design objects to them.

So this fork exists to move faster on an agreed direction, not to depart from one. Worth remembering
when a design decision comes up: agreeing with upstream's architecture is usually also the cheaper
choice, because it keeps `git merge upstream/main` boring.

## Build it generically anyway

Not out of deference — out of self-interest. A hook runner that hardcodes one workstation's bootstrap
script, or a runtime adapter shaped around one terminal, has to be rewritten the first time a second
case appears, and it makes every future upstream merge worse.

Practical form:

1. Identify the missing primitive.
2. Implement it generically, in the layer it belongs to.
3. Wire this workstation's specifics on top of it, separately.
4. Offer the primitive upstream if and when that is worth the round trip.

Step 3 is the one that gets skipped, and skipping it is what turns a fork into a maintenance burden.

## Generic enough to offer upstream

Everything in [roadmap](../kra/roadmap.md) phases 1–3 is broadly useful and encodes no single user's
workflow, so none of it needs to be fork-only:

- bounded worker pool for fan-out operations,
- lifecycle hooks with a documented environment contract,
- hook trust/approval,
- background hooks with process tracking,
- `ws status --format json`,
- `ws exec`,
- repository `focus`,
- explicit workspace readiness state,
- workspace trash and GC.

Phase 4.1 (runtime port rename) is generic but broad — ~27 files. Do it as one intentional change
rather than letting it accumulate as scattered drift, whether or not it is ever offered upstream.

## Keep outside the tool

- OpenSpec-specific behavior,
- AXIS-specific behavior,
- agent assignment policy,
- personal repository grouping,
- workstation-only conventions.

These belong in a higher integration layer or in configuration — not because upstream would object,
but because they are single-user behavior that will never be worth maintaining inside a general tool.

`kra` currently has no command-level extension point
([gap analysis §14](../kra/gap-analysis.md#14-custom-subcommands--missing-but-the-pattern-exists)), so
this rule has nowhere to land until
[roadmap 4.3](../kra/roadmap.md#43-custom-subcommand-extension-point-upstreamable) exists. That is an
argument for building 4.3 reasonably early: it is the pressure valve that keeps workstation specifics
out of the substrate.

## The project's own conventions

Read from `AGENTS.md`, `docs/dev/backlog/README.md`, and `docs/dev/spec/concepts/architecture.md`.
These are inherited with the fork, and worth following even for fork-only work — they are what keeps
the codebase legible and upstream merges boring:

- **Spec first.** Behavior is specified under `docs/dev/spec/` before it is implemented, and a spec
  is flipped to `status: implemented` as part of the done definition. Ambiguity is resolved by
  changing the spec, not by deciding it in code.
- **Backlog-shaped work.** Work is tracked as tickets under `docs/dev/backlog/<EPIC>.md` with
  `What` / `Specs` / `Depends` / `Serial` fields, and the index's `File Status` is kept in sync. A
  hooks proposal most naturally arrives as a new epic file with linked specs.
- **Layering is enforced by tests.** `internal/archguard/layering_test.go` asserts the dependency
  direction (`cli` → `app` → `domain`, adapters in `infra`/`ui`) and holds an explicit allowlist for
  direct infra imports under `cli` that "no silent expansion" is permitted against. Hook execution
  belongs in `app` over an `infra` port, not in `cli`.
- **Quality gate before any commit worth keeping:** `test -z "$(gofmt -l .)"`,
  `./scripts/lint-ui-color.sh`, `go vet ./...`, `go test ./...`. Note that GitHub Actions does not run
  on this fork, so this gate is local-only — nothing will catch it for you.
- **Commit scope discipline.** Commits are sliced to one item and stage only intended paths. The same
  allowlist rule governs `kra` commands that commit inside `KRA_ROOT`.
- If an upstream issue or PR is ever worth opening, note that the maintainer works in Japanese
  (`AGENTS.md` sets Japanese as the response language).

## Fork hygiene

- Keep generic primitives in separate commits from their workstation-specific callers. This is the
  one habit that preserves every future option at almost no cost.
- Track compatibility promises in `docs/dev/spec/core/COMPATIBILITY.md` when metadata schemas change —
  `.kra.meta.json` is read by workspaces that outlive any single version.
- Merge `upstream/main` periodically rather than in one large catch-up. The zero-divergence position
  above holds only until the first fork-local commit lands.
