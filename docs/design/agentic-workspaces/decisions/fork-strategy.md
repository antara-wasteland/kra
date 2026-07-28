---
title: "Fork and Upstream Strategy"
status: draft
---

# Fork and Upstream Strategy

This repository is a fork of `tasuku43/kra`. This page records when that fork should carry a change
and when the change should go upstream instead.

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
five positions. Only lifecycle extensibility is absent — and absent as a missing feature, not as a
rejected one. That is a weak case for divergence and a strong case for contribution.

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
([gap analysis §14](../kra/gap-analysis.md#14-custom-subcommands--missing)), so this rule has nowhere
to land until [roadmap 4.3](../kra/roadmap.md#43-custom-subcommand-extension-point-upstreamable)
exists. Until then, resist putting workstation specifics into the fork — that is exactly how a fork
becomes unmergeable.

## Fork hygiene

- Keep fork-local commits separable from upstreamable ones; do not mix a generic primitive and its
  workstation-specific caller in one commit.
- Follow this repo's spec-then-implementation convention (`docs/dev/spec/`) even for fork-local work,
  so an upstream PR does not need the spec written retroactively.
- Track compatibility promises in `docs/dev/spec/core/COMPATIBILITY.md` when metadata schemas change.
