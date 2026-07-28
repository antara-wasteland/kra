---
title: "Fork and Upstream Strategy"
status: draft
---

# Fork and Upstream Strategy

This repository is a fork of `tasuku43/kra`. This page records when that fork should carry a change
and when the change should go upstream instead.

## Current state: zero divergence

`rishwanth-th/kra` `main` is identical to `tasuku43/kra` `main` at `1fdcd0b` — no commits ahead, none
behind. Everything the fork can do, upstream does; there is no fork-local behavior to maintain and
nothing to reconcile.

That is the cheapest possible position and worth keeping deliberately. Every roadmap item is a
green-field contribution decision rather than a divergence already incurred.

## Do not fork merely to customize defaults

Prefer configuration or small upstream PRs for:

- path and branch templates,
- runtime adapters,
- hook events,
- JSON output coverage,
- custom commands.

## Fork when the architectural unit differs

A fork is justified when upstream fundamentally does not want:

- task-first workspaces,
- polyrepo composition,
- lifecycle extensibility,
- runtime abstraction,
- machine-first operation.

These are architectural choices, not cosmetic preferences.

The [gap analysis](../kra/gap-analysis.md) found that upstream `kra` already holds four of those
five positions. Only lifecycle extensibility is absent, and upstream's own backlog shows it absent as
an unfilled gap rather than a rejected proposal: all 17 backlog epic files are fully checked off, and
none of them ever raised hooks. Nothing in `kra`'s stated design objects to them.

That is a weak case for divergence and a strong case for contribution.

## Preferred sequence

1. Use upstream.
2. Identify the missing primitive.
3. Implement it generically.
4. Raise an upstream PR.
5. Maintain fork-local divergence only where upstream philosophy and workstation requirements
   genuinely differ.

Step 3 is the one that gets skipped. A hook runner that hardcodes this workstation's bootstrap
script, or a runtime adapter shaped around one terminal, is not upstreamable and quietly commits the
fork to permanent maintenance.

## Upstream candidates

Everything in [roadmap](../kra/roadmap.md) phases 1–3 is broadly useful and encodes no single user's
workflow:

- bounded worker pool for fan-out operations,
- lifecycle hooks with a documented environment contract,
- hook trust/approval,
- background hooks with process tracking,
- `ws status --format json`,
- `ws exec`,
- repository `focus`,
- explicit workspace readiness state,
- workspace trash and GC.

Phase 4.1 (runtime port rename) is upstreamable but broad; raise it as one intentional change rather
than letting it accumulate as fork drift.

## Keep outside the tool

- OpenSpec-specific behavior,
- AXIS-specific behavior,
- agent assignment policy,
- personal repository grouping,
- workstation-only conventions.

These belong in a higher integration layer or in configuration. Note that `kra` currently has no
custom-subcommand extension point
([gap analysis §14](../kra/gap-analysis.md#14-custom-subcommands--missing-but-the-pattern-exists)),
so this rule has nowhere to land until
[roadmap 4.3](../kra/roadmap.md#43-custom-subcommand-extension-point-upstreamable) exists. Until
then, resist putting workstation specifics into the fork — that is exactly how a fork becomes
unmergeable.

## Contributing upstream: what the project expects

Read from `AGENTS.md`, `docs/dev/backlog/README.md`, and `docs/dev/spec/concepts/architecture.md`.
A change that ignores these will not land regardless of its merit:

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
- **Quality gate before any PR:** `test -z "$(gofmt -l .)"`, `./scripts/lint-ui-color.sh`,
  `go vet ./...`, `go test ./...`.
- **Commit scope discipline.** Commits are sliced to one backlog item and stage only intended paths.
  The same allowlist rule governs commands that commit inside `KRA_ROOT`.
- The maintainer works in Japanese (`AGENTS.md` sets Japanese as the response language). Worth
  knowing when writing an issue or PR description.

## Fork hygiene

- Keep fork-local commits separable from upstreamable ones; do not mix a generic primitive and its
  workstation-specific caller in one commit.
- Follow the spec-then-implementation convention even for fork-local work, so an upstream PR does not
  need the spec written retroactively.
- Track compatibility promises in `docs/dev/spec/core/COMPATIBILITY.md` when metadata schemas change.
- Re-check divergence before starting work — the zero-divergence position above is only true until
  the first fork-local commit lands on `main`.
