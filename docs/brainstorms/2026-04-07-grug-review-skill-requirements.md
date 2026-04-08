---
date: 2026-04-07
topic: grug-review-skill
---

# Grug Review Skill

## Problem Frame

Code reviews catch bugs and style issues but rarely challenge whether the code is *unnecessarily complex*. grugbrain.dev articulates a philosophy — fight complexity, resist premature abstraction, prefer simplicity — that many developers agree with but don't systematically apply during review. A Claude Code skill that reviews diffs through grug's lens would surface complexity problems that standard linters and reviewers miss, in a voice that's memorable and disarming.

## Requirements

- R1. The skill runs a multi-agent review pipeline inspired by ce:review's architecture: always-on core reviewers + conditional reviewers triggered by diff content.
- R2. Always-on reviewers (run on every review):
  - **grug-complexity**: overall complexity, over-engineering, unnecessary features, YAGNI violations
  - **grug-abstraction**: premature factoring, too many layers, Chesterton's fence violations, single-use abstractions
  - **grug-expression**: nested conditionals, unnamed intermediates, hard-to-follow logic, expressions that resist debugging
  - **grug-locality**: code spread across too many files, separation of concerns overuse, locality of behavior violations
- R3. Conditional reviewers (triggered by diff content):
  - **grug-testing**: when test files are touched — mocking excess, unit-vs-integration balance, missing regression tests
  - **grug-type-shaman**: when type definitions or generics are touched — generics abuse, over-abstracted type hierarchies, "astral projection" types
  - **grug-concurrency**: when async/threading/concurrency primitives appear — unnecessary complexity, simpler model available
  - **grug-dry**: when similar code patterns appear or abstractions are introduced — is the DRY refactor actually simpler than the repetition it eliminates?
- R4. Each reviewer returns structured findings that are merged and deduplicated by the orchestrator, borrowing ce:review's fingerprint-based dedup (file + line bucket + normalized title).
- R5. Findings use a 3-level severity scale with grug-flavored text labels (no emojis): `bad`, `very bad`, `complexity demon`.
- R6. Output uses structured tables with a grug-voice commentary column ("Grug Say"). Standard columns: #, severity, file, issue title, grug's take + suggestion.
- R7. Grug also calls out things he likes — simple code, good logging, locality of behavior. Positive findings appear in a separate "grug like" section.
- R8. Each finding includes a suggested fix in grug voice, not just a complaint.
- R9. The review ends with a grug-voice verdict summarizing overall impression (e.g., "grug mass mass confuse" or "grug approve, complexity demon not welcome here").
- R10. Confidence gating: suppress findings below a threshold to avoid noise. Reviewers should only flag things they're confident about — grug hates false alarms.

## Success Criteria

- Running `/grug` on a diff produces a scannable, entertaining review that surfaces real complexity problems a standard review would miss.
- The grug voice is consistent and recognizable across all reviewers — it reads like one opinionated caveman, not eight different bots.
- False positive rate is low enough that developers don't learn to ignore findings.

## Scope Boundaries

- No autofix mode — grug suggests, developer decides.
- No integration with CI/GitHub comments — this is a local CLI review skill.
- The skill does not replace ce:review — it's a complementary opinionated lens focused on complexity/simplicity.
- No custom configuration for which reviewers to enable — the always-on/conditional split handles this automatically.

## Key Decisions

- **Multi-agent over single agent**: Chosen for structured coverage of grug's distinct opinions, even though grug himself would disapprove of the complexity. The irony is noted and accepted.
- **Structured tables + grug commentary over pure grug prose**: Scannable output matters more than full immersion. The "Grug Say" column carries the voice.
- **3-level severity over 4**: Simpler, and grug doesn't do nuance. `bad` / `very bad` / `complexity demon`.
- **Praise included**: Grug calls out simplicity he likes. Makes criticism land harder by contrast and keeps the tone balanced.
- **Text severity labels, no emojis**: Per user preference.

## Outstanding Questions

### Deferred to Planning

- [Affects R1][Technical] What's the right confidence threshold for grug reviewers? ce:review uses 0.60 — should grug be higher (less noise) or the same?
- [Affects R4][Needs research] How should the orchestrator prompt be structured? Study ce:review's SKILL.md for the 6-stage pipeline and decide which stages to keep/simplify.
- [Affects R2, R3][Technical] What should each persona's prompt look like? Need to define identity, calibration, and suppress conditions for each of the 8 reviewers.
- [Affects R6][Technical] Exact output template — header, tables, verdict format. Should reference ce:review's output template as starting point.
- [Affects R1][Technical] Should the skill accept a PR number, branch, or just review the current diff? ce:review supports all three — decide which subset grug needs.

## Next Steps

-> `/ce:plan` for structured implementation planning
