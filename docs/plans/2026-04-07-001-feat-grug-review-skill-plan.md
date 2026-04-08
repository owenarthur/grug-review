---
title: "feat: Add grug review skill — complexity-focused code review in grugbrain voice"
type: feat
status: completed
date: 2026-04-07
origin: docs/brainstorms/2026-04-07-grug-review-skill-requirements.md
---

# feat: Add grug review skill

## Overview

Build a Claude Code skill invoked via `/grug` that reviews code diffs through the lens of grugbrain.dev philosophy. Uses a multi-agent pipeline (inspired by ce:review) with 4 always-on and 4 conditional reviewer personas, all speaking in grug's caveman voice. Outputs structured tables with grug commentary, praise for simplicity, and a grug-voice verdict.

## Problem Frame

Code reviews catch bugs and style issues but rarely challenge whether code is *unnecessarily complex*. grugbrain.dev articulates a philosophy — fight complexity, resist premature abstraction, prefer simplicity — that developers agree with but don't systematically apply. This skill makes that philosophy actionable during review. (see origin: docs/brainstorms/2026-04-07-grug-review-skill-requirements.md)

## Requirements Trace

- R1. Multi-agent review pipeline: always-on + conditional reviewers
- R2. Always-on: grug-complexity, grug-abstraction, grug-expression, grug-locality
- R3. Conditional: grug-testing, grug-type-shaman, grug-concurrency, grug-dry
- R4. Structured findings merged with fingerprint-based dedup
- R5. 3-level severity: `bad`, `very bad`, `complexity demon` (no emojis)
- R6. Structured tables with "Grug Say" column
- R7. Positive findings in separate "grug like" section
- R8. Each finding includes a suggested fix in grug voice
- R9. Grug-voice verdict at the end
- R10. Confidence gating to suppress noise

## Scope Boundaries

- No autofix mode — grug suggests, developer decides (see origin)
- No CI/GitHub comment integration — local CLI review only (see origin)
- Complementary to ce:review, not a replacement (see origin)
- No custom configuration for reviewer selection (see origin)

## Context & Research

### Relevant Code and Patterns

ce:review's architecture provides the blueprint. Key patterns to adapt:

- **SKILL.md as orchestrator**: Single file defines the pipeline, references shared materials
- **`agents/review/*.md` persona format**: Frontmatter (name, description, model, tools, color) + identity + hunting list + confidence calibration + suppress list + output schema
- **`references/` directory**: findings-schema.json, subagent-template.md, diff-scope.md, output-template.md
- **Subagent template with variable injection**: `{persona_file}`, `{diff_scope_rules}`, `{schema}`, `{intent_summary}`, `{file_list}`, `{diff}`
- **Fingerprint dedup**: `normalize(file) + line_bucket(line, ±3) + normalize(title)`
- **Confidence calibration per persona**: 3-tier (high/moderate/low) with explicit suppress conditions

### Institutional Learnings

No prior `docs/solutions/` exist — this is a greenfield project.

### External References

- grugbrain.dev — the source philosophy. Every persona's opinions derive from specific sections of this page.
- ce:review SKILL.md (~37KB) — the architectural reference for pipeline stages, persona spawning, merge logic, and output formatting.

## Key Technical Decisions

- **Confidence threshold: 0.70** (vs ce:review's 0.60): Grug hates noise. Higher threshold means fewer but more confident findings. Grug's domain (complexity/abstraction) is more subjective than correctness/security, so a higher bar prevents speculative complaints.
- **Simplified pipeline: 4 stages** (vs ce:review's 6): Drop autofix routing and post-review fix loops entirely. Stage 1: scope, Stage 2: select, Stage 3: spawn, Stage 4: present.
- **Simplified findings schema**: Drop `autofix_class`, `owner`, `requires_verification` fields. Add `grug_say` (grug-voice commentary) and `suggestion` (grug-voice fix suggestion). Add `positive` boolean for "grug like" findings.
- **Shared voice guide**: A `grug-voice-guide.md` reference file included in every persona prompt. Defines grug's speaking patterns, vocabulary, recurring phrases, and tone rules. This is the key mechanism for making 8 agents sound like one caveman.
- **Scope: diff-only, no PR checkout**: Support current uncommitted changes (default) or branch vs detected base. No `gh pr checkout` — keeps the skill simple and avoids mutation.
- **No modes**: Single mode (interactive report). No autofix, no report-only. Grug is simple.
- **Skill lives at project root**: `skills/grug/SKILL.md` + `skills/grug/references/` + `agents/grug/*.md`. Standard plugin layout so it can be published or used locally.

## Open Questions

### Resolved During Planning

- **Confidence threshold?** → 0.70. Grug's domain is subjective; higher bar prevents speculative noise.
- **Which ce:review stages to keep?** → 4 of 6. Drop post-review fix loop and autofix routing. Keep scope detection, reviewer selection, parallel spawn, merge+present.
- **Scope input?** → Current diff (default) or branch name. No PR checkout.
- **Where do files live?** → `skills/grug/` for skill, `agents/grug/` for personas. Standard plugin layout.

### Deferred to Implementation

- **Exact grug vocabulary list**: The voice guide needs specific recurring phrases, but the best way to tune these is iteratively during implementation.
- **Conditional trigger heuristics**: Exact rules for when grug-testing, grug-type-shaman, etc. activate. The orchestrator reads the diff and decides — specific patterns will emerge during implementation.

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.*

```
                         /grug [branch]
                              │
                    ┌─────────▼──────────┐
                    │  Stage 1: Scope    │
                    │  git diff → files, │
                    │  diff content      │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Stage 2: Select   │
                    │  Always-on (4) +   │
                    │  Conditional (0-4) │
                    └─────────┬──────────┘
                              │
              ┌───────┬───────┼───────┬───────┐
              ▼       ▼       ▼       ▼       ▼
          ┌───────┐┌──────┐┌──────┐┌──────┐┌──────┐
          │complex││abstr.││expr. ││local.││cond. │
          │-ity   ││      ││      ││      ││(0-4) │
          └───┬───┘└──┬───┘└──┬───┘└──┬───┘└──┬───┘
              │       │       │       │       │
              └───────┴───────┼───────┴───────┘
                              │
                    ┌─────────▼──────────┐
                    │  Stage 4: Merge    │
                    │  Dedup, gate,      │
                    │  sort, present     │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Output: Tables,   │
                    │  "grug like",      │
                    │  verdict           │
                    └────────────────────┘
```

**Findings flow:**
- Each persona returns JSON: `{ reviewer, findings[], positives[] }`
- Orchestrator dedup by fingerprint: `normalize(file) + line_bucket(±3) + normalize(title)`
- Gate: suppress findings with confidence < 0.70
- Partition: negative findings → severity-grouped table, positive findings → "grug like" section
- Sort: `complexity demon` first → `very bad` → `bad`, then confidence desc → file → line
- Verdict: synthesize overall impression in grug voice

## Implementation Units

- [ ] **Unit 1: Project scaffolding and skill registration**

  **Goal:** Create the directory structure and empty skill/agent files so the project has shape.

  **Requirements:** R1

  **Dependencies:** None

  **Files:**
  - Create: `skills/grug/SKILL.md` (placeholder)
  - Create: `skills/grug/references/findings-schema.json`
  - Create: `skills/grug/references/grug-voice-guide.md`
  - Create: `skills/grug/references/subagent-template.md`
  - Create: `skills/grug/references/diff-scope.md`
  - Create: `skills/grug/references/output-template.md`
  - Create: `agents/grug/` directory (empty, populated in later units)

  **Approach:**
  - Mirror ce:review's directory convention: `skills/<name>/SKILL.md` + `skills/<name>/references/` + `agents/<name>/`
  - Start with the findings schema and voice guide since all other files depend on them

  **Patterns to follow:**
  - ce:review directory layout
  - ce:review findings-schema.json structure (simplified: drop autofix fields, add grug-specific fields)

  **Test scenarios:**
  - All files exist at expected paths
  - findings-schema.json is valid JSON Schema

  **Verification:**
  - Directory tree matches the planned layout
  - Schema parses without errors

- [ ] **Unit 2: Findings schema and voice guide**

  **Goal:** Define the JSON contract for reviewer output and the shared grug voice reference that ensures all 8 personas sound like one caveman.

  **Requirements:** R4, R5, R6, R7, R8, R10

  **Dependencies:** Unit 1

  **Files:**
  - Modify: `skills/grug/references/findings-schema.json`
  - Modify: `skills/grug/references/grug-voice-guide.md`

  **Approach:**
  - **findings-schema.json**: Adapt ce:review's schema. Keep: `reviewer`, `findings[]`, `title`, `severity` (enum: `bad`, `very bad`, `complexity demon`), `file`, `line`, `confidence` (0.0-1.0), `evidence[]`. Add: `grug_say` (string — grug voice commentary + suggestion combined), `positive` (boolean — true for "grug like" findings). Drop: `autofix_class`, `owner`, `requires_verification`, `why_it_matters` (replaced by `grug_say`), `suggested_fix` (folded into `grug_say`).  Keep: `residual_risks[]` (renamed to `things_grug_not_sure_about`), drop `testing_gaps[]` (grug-testing persona handles this in findings).
  - **grug-voice-guide.md**: Define grug's speaking patterns extracted from grugbrain.dev:
    - Lowercase, minimal punctuation, short sentences
    - Third person ("grug see", "grug think", not "I see")
    - Recurring phrases: "complexity demon", "big brain", "shiney rock", "mass mass confuse"
    - Tone: direct, honest, slightly self-deprecating, never cruel
    - Anti-patterns: no corporate speak, no hedging, no "consider perhaps maybe"
    - Praise patterns: "grug like", "grug approve", "no complexity demon here"
    - Severity voice: `bad` = mild grumble, `very bad` = genuine concern, `complexity demon` = alarm

  **Patterns to follow:**
  - ce:review findings-schema.json structure and JSON Schema draft-07 format
  - grugbrain.dev voice and recurring phrases

  **Test scenarios:**
  - Schema validates a sample finding with all required fields
  - Schema rejects a finding missing `grug_say`
  - Voice guide covers all recurring phrases from grugbrain.dev

  **Verification:**
  - Schema is valid JSON Schema that can validate reviewer output
  - Voice guide is specific enough to produce consistent voice across personas

- [ ] **Unit 3: Diff scope rules and subagent template**

  **Goal:** Define how reviewers understand what they're reviewing and the prompt template used to spawn each persona.

  **Requirements:** R1, R4

  **Dependencies:** Unit 2

  **Files:**
  - Modify: `skills/grug/references/diff-scope.md`
  - Modify: `skills/grug/references/subagent-template.md`

  **Approach:**
  - **diff-scope.md**: Simplify ce:review's 3-tier system. Keep primary (changed lines) and pre-existing (unchanged, unrelated) tiers. Drop secondary tier — grug reviews are about complexity patterns, not subtle interaction bugs. Pre-existing complexity issues are still worth noting but reported separately.
  - **subagent-template.md**: Adapt ce:review's template. Variables: `{persona_file}`, `{voice_guide}`, `{diff_scope_rules}`, `{schema}`, `{file_list}`, `{diff}`. Key addition: inject the voice guide as a `<voice>` block so every persona gets the same grug speaking patterns. Template should emphasize: return ONLY valid JSON, suppress below 0.70 confidence, include positive findings when code is genuinely simple, read-only operations only.

  **Patterns to follow:**
  - ce:review diff-scope.md (simplified)
  - ce:review subagent-template.md variable injection pattern

  **Test scenarios:**
  - Template produces a coherent prompt when variables are substituted
  - Scope rules clearly distinguish primary from pre-existing findings

  **Verification:**
  - Template has all variable slots documented
  - Scope rules are unambiguous

- [ ] **Unit 4: Always-on persona definitions (4 agents)**

  **Goal:** Create the 4 always-on grug reviewer personas with distinct scopes, grug-appropriate opinions, and confidence calibration.

  **Requirements:** R2, R5, R8, R10

  **Dependencies:** Unit 2, Unit 3

  **Files:**
  - Create: `agents/grug/grug-complexity.md`
  - Create: `agents/grug/grug-abstraction.md`
  - Create: `agents/grug/grug-expression.md`
  - Create: `agents/grug/grug-locality.md`

  **Approach:**
  - Follow ce:review agent format: frontmatter + identity + hunting list + confidence calibration + suppress list + output format
  - Each persona gets `tools: Read, Grep, Glob, Bash` (read-only)
  - Each persona's hunting list derived from specific grugbrain.dev sections:
    - **grug-complexity**: "complexity very very bad", unnecessary features, over-engineering, YAGNI, "80/20 solution" — hunts for: God classes, methods doing too many things, unnecessary config/options, features nobody asked for, deep call chains
    - **grug-abstraction**: premature factoring, "wait until natural cut-points emerge", Chesterton's fence — hunts for: single-use abstractions, abstract factories for one implementation, premature interfaces, layers that just delegate, removing code without understanding why it exists
    - **grug-expression**: "break expressions into named intermediate variables" — hunts for: nested ternaries, multi-line boolean expressions, chained method calls that resist debugging, unnamed magic values, conditionals where you can't set a breakpoint on each part
    - **grug-locality**: "Locality of Behavior", code on the thing it does — hunts for: behavior spread across many files for one concept, excessive indirection (click through 5 files to understand one button), separation of concerns that hurts understanding
  - Confidence calibration at 0.70 threshold:
    - High (0.85+): pattern is unambiguous (e.g., 4-level nested ternary, abstract factory with one implementation)
    - Moderate (0.70-0.84): pattern present but context could justify it
    - Low (<0.70): suppress — could be justified, grug not sure
  - Each persona's suppress list explicitly excludes the other personas' domains

  **Patterns to follow:**
  - ce:review correctness-reviewer.md and maintainability-reviewer.md format
  - grugbrain.dev specific sections for each domain

  **Test scenarios:**
  - Each persona has non-overlapping scope with the others
  - Suppress conditions prevent one persona from flagging another's domain
  - Confidence calibration is specific enough to guide findings
  - Voice is consistent with grug-voice-guide.md

  **Verification:**
  - All 4 files follow identical structure
  - No two personas hunt for the same thing
  - Each persona references specific grugbrain.dev opinions

- [ ] **Unit 5: Conditional persona definitions (4 agents)**

  **Goal:** Create the 4 conditional grug reviewer personas that activate based on diff content.

  **Requirements:** R3, R5, R8, R10

  **Dependencies:** Unit 2, Unit 3

  **Files:**
  - Create: `agents/grug/grug-testing.md`
  - Create: `agents/grug/grug-type-shaman.md`
  - Create: `agents/grug/grug-concurrency.md`
  - Create: `agents/grug/grug-dry.md`

  **Approach:**
  - Same format as always-on personas but with activation guidance in the description
  - Each persona's hunting list derived from specific grugbrain.dev sections:
    - **grug-testing**: "integration tests sweet spot", avoid "first test methodology", excessive mocking, always write regression tests for bugs — hunts for: tests that mock everything, unit tests for trivial getters, missing regression test when fixing a bug, test setup longer than test itself
    - **grug-type-shaman**: "type system shamans pursuing abstraction-heavy designs", "generics especially dangerous — limit to container classes" — hunts for: generics beyond containers, type hierarchies deeper than 2 levels, mapped/conditional types that require a PhD, types that serve the type system not the developer
    - **grug-concurrency**: "fear concurrency as much as possible", prefer stateless handlers + simple job queues — hunts for: unnecessary async/await, manual thread management when a job queue works, shared mutable state, complex locking when optimistic concurrency suffices
    - **grug-dry**: "grug values simple repeated code over elaborate callbacks, closures, or complex object models" — hunts for: abstractions that are harder to understand than the duplication they eliminate, callback/closure patterns replacing straightforward code, 3 lines repeated twice refactored into a 15-line helper
  - Activation triggers (for orchestrator to evaluate):
    - grug-testing: diff touches files matching `*test*`, `*spec*`, `*_test.*`, `*.test.*`
    - grug-type-shaman: diff touches type definition files, contains `generic`, `<T>`, `interface`, `type alias`, or complex type annotations
    - grug-concurrency: diff contains `async`, `await`, `Promise`, `Thread`, `Mutex`, `Lock`, `Channel`, `goroutine`, `spawn`, `tokio`
    - grug-dry: diff introduces new helper/utility functions, or the orchestrator notices similar code patterns in the diff

  **Patterns to follow:**
  - ce:review adversarial-reviewer.md format (conditional persona)
  - grugbrain.dev specific sections for each domain

  **Test scenarios:**
  - Each persona activates only when relevant diff content is present
  - Hunting lists are specific to the domain and don't overlap with always-on personas
  - Voice is consistent with grug-voice-guide.md

  **Verification:**
  - All 4 files follow identical structure to always-on personas
  - Activation triggers are documented clearly enough for the orchestrator to evaluate

- [ ] **Unit 6: Output template**

  **Goal:** Define the exact output format for the final grug review report.

  **Requirements:** R5, R6, R7, R9

  **Dependencies:** Unit 2

  **Files:**
  - Modify: `skills/grug/references/output-template.md`

  **Approach:**
  - Adapt ce:review's output template to grug's format. Structure:
    - **Header**: scope (X files, Y lines), reviewers spawned (list with one-line justifications for conditionals)
    - **Findings table**: grouped by severity (`complexity demon` first, then `very bad`, then `bad`). Columns: `#`, `Sev`, `File`, `Issue`, `Grug Say`. Omit empty severity groups.
    - **grug like**: separate section for positive findings. Simple bullet list: `- [file:line] grug say text`. Omit section if no positives.
    - **Pre-existing**: separate table for complexity issues in unchanged code. Informational only.
    - **Coverage**: count of suppressed findings (below 0.70), reviewers that found nothing
    - **Verdict**: blockquote in grug voice. Synthesize overall impression. Examples: "grug mass mass confuse. complexity demon have big party in this code." or "grug approve. simple code, clear intent, no complexity demon. grug put on wall of cave as example for young grug."
  - Use pipe-delimited markdown tables (not ASCII box-drawing), matching ce:review convention

  **Patterns to follow:**
  - ce:review review-output-template.md structure (simplified)

  **Test scenarios:**
  - Template handles: all findings negative, all positive, mix, no findings at all
  - Severity groups omitted when empty
  - Verdict examples cover the range from "terrible" to "great"

  **Verification:**
  - Template renders correctly as markdown
  - All sections have clear inclusion/exclusion rules

- [ ] **Unit 7: SKILL.md orchestrator**

  **Goal:** Write the main orchestrator that ties everything together — the 4-stage pipeline that scopes the diff, selects reviewers, spawns them, merges findings, and presents the report.

  **Requirements:** R1, R4, R10

  **Dependencies:** Units 1-6

  **Files:**
  - Modify: `skills/grug/SKILL.md`

  **Approach:**
  - Frontmatter: `name: grug`, `description: "Code review through the lens of grugbrain.dev..."`, `argument-hint: "[branch name]"`
  - Include references: `@./references/grug-voice-guide.md`, `@./references/subagent-template.md`, `@./references/diff-scope.md`
  - **Stage 1 — Scope**: Detect diff source. If branch argument provided, `git diff $(git merge-base HEAD <branch>)..<branch>`. Otherwise, `git diff HEAD` for uncommitted changes, falling back to `git diff $(git merge-base origin/HEAD HEAD)..HEAD` for committed-but-unpushed changes. Collect file list and diff content.
  - **Stage 2 — Select reviewers**: Always spawn 4 always-on personas. For each conditional persona, read the diff and decide whether to activate based on content (not keyword matching — use judgment, like ce:review). Announce selected team with one-line justifications for each conditional.
  - **Stage 3 — Spawn**: Use subagent template to spawn all selected personas in parallel via Agent tool. Each gets: persona file content, voice guide, diff scope rules, schema, file list, diff. Collect JSON responses.
  - **Stage 4 — Merge and present**:
    - Validate JSON from each reviewer
    - Confidence gate: suppress findings with confidence < 0.70, count suppressed
    - Dedup: fingerprint = `normalize(file) + line_bucket(line, ±3) + normalize(title)`. On collision: keep highest severity, highest confidence, combine grug_say from both, note both reviewers
    - Partition: negative findings → severity table, positive findings → "grug like" section, pre-existing → separate table
    - Sort: severity desc → confidence desc → file → line
    - Render using output template
    - Write verdict in grug voice synthesizing overall impression
  - **Quality gate before output**: Check that findings are code-grounded (have evidence), voice is consistent, no findings below confidence threshold leaked through

  **Patterns to follow:**
  - ce:review SKILL.md 6-stage pipeline (stages 1, 3, 4, 5+6 mapped to our stages 1-4)
  - ce:review's reviewer selection approach (judgment, not keyword matching)
  - ce:review's dedup fingerprint algorithm
  - ce:review's parallel Agent spawning pattern

  **Test scenarios:**
  - Empty diff produces "grug see nothing to review" message
  - Diff with only simple code produces only "grug like" findings
  - Diff with complexity issues produces severity-sorted table
  - Conditional reviewers correctly activate/skip based on diff content
  - Duplicate findings from multiple reviewers are merged, not duplicated
  - Suppressed findings are counted but not shown

  **Verification:**
  - Running `/grug` on a diff produces the expected output format
  - All 4 stages execute in order
  - Parallel spawning works for all selected reviewers
  - Verdict reflects the overall findings accurately

## System-Wide Impact

- **Interaction graph:** The skill spawns Agent sub-agents that use Read, Grep, Glob, Bash tools in read-only mode. No file mutations, no git mutations.
- **Error propagation:** If a persona agent fails or returns invalid JSON, the orchestrator should skip that reviewer and note it in the coverage section. Never crash the whole review.
- **State lifecycle risks:** None — the skill is stateless. No artifacts written, no run IDs, no persistent state.
- **API surface parity:** N/A — this is a local CLI skill only.
- **Integration coverage:** The main integration point is the Agent tool for spawning sub-agents. The skill depends on git being available for diff computation.

## Risks & Dependencies

- **Voice consistency risk**: 8 different agents may drift in voice despite the shared guide. Mitigation: the voice guide must be very specific with examples, and the subagent template must inject it prominently.
- **Noise risk**: Complexity/abstraction reviews are inherently subjective. Mitigation: 0.70 confidence threshold + specific suppress conditions per persona.
- **Overlap risk**: grug-complexity and grug-abstraction have adjacent domains. Mitigation: explicit scope boundaries in each persona's suppress list + dedup in merge stage.
- **Dependency**: Requires Claude Code's Agent tool for parallel sub-agent spawning. If not available, sequential fallback needed (noted in SKILL.md).

## Sources & References

- **Origin document:** [docs/brainstorms/2026-04-07-grug-review-skill-requirements.md](docs/brainstorms/2026-04-07-grug-review-skill-requirements.md)
- **Architecture reference:** ce:review SKILL.md and agents/review/ persona definitions
- **Philosophy source:** grugbrain.dev
- **Key ce:review files studied:** findings-schema.json, subagent-template.md, diff-scope.md, review-output-template.md, correctness-reviewer.md, adversarial-reviewer.md, maintainability-reviewer.md
