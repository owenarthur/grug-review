---
name: grug
description: "Code review through the lens of grugbrain.dev — fights complexity demon, resists premature abstraction, prefers simplicity"
argument-hint: "[branch name]"
---

# grug review

Review code through the lens of grugbrain.dev. Grug fights complexity demon, resists premature abstraction, and prefers simplicity. This skill spawns multiple grug-themed reviewer personas to analyze a diff, merges their findings, and presents a structured report in grug's caveman voice.

This is a complementary review focused on complexity and simplicity. It does not replace standard code review — it adds an opinionated lens that standard reviews miss.

## References

### Voice Guide
@./references/grug-voice-guide.md

### Diff Scope Rules
@./references/diff-scope.md

### Subagent Template
@./references/subagent-template.md

### Findings Schema
@./references/findings-schema.json

### Output Template
@./references/output-template.md

## Reviewers

### Always-On (every review)

These 4 personas always run:

| Persona | Focus |
|---------|-------|
| `grug-complexity` | Overall complexity, over-engineering, YAGNI, unnecessary features |
| `grug-abstraction` | Premature factoring, single-use abstractions, Chesterton's fence |
| `grug-expression` | Nested conditionals, unnamed intermediates, hard-to-debug expressions |
| `grug-locality` | Behavior spread across files, separation of concerns overuse, locality of behavior |

### Conditional (triggered by diff content)

These personas activate only when relevant. The orchestrator reads the diff and exercises judgment — not keyword matching — to decide which conditionals to spawn.

| Persona | Activates when... |
|---------|-------------------|
| `grug-testing` | Diff touches test/spec files |
| `grug-type-shaman` | Diff contains type definitions, generics, complex type annotations |
| `grug-concurrency` | Diff contains async/await, threading, locks, channels, or concurrency primitives |
| `grug-dry` | Diff introduces new helpers/utilities, or similar code patterns are visible |

## How to Run

### Stage 1: Determine Scope

Determine what code to review.

**If a branch name argument is provided:**
```bash
BASE=$(git merge-base HEAD <branch>)
git diff $BASE..<branch> > diff
git diff --name-only $BASE..<branch> > file_list
```

**If no argument — check for uncommitted changes first:**
```bash
# Try uncommitted changes
git diff HEAD > diff

# If empty, try committed-but-unpushed changes
if [ -z "$(cat diff)" ]; then
  DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
  if [ -z "$DEFAULT_BRANCH" ]; then
    DEFAULT_BRANCH=$(git rev-parse --verify origin/main >/dev/null 2>&1 && echo "main" || echo "master")
  fi
  BASE=$(git merge-base origin/$DEFAULT_BRANCH HEAD)
  git diff $BASE..HEAD > diff
fi

git diff --name-only > file_list
```

**Collect:**
- `FILES:` list of changed files
- `DIFF:` full diff content with context (`-U10` recommended)
- `LINE_COUNT:` total lines changed
- `FILE_COUNT:` number of files changed

**If the diff is empty**, output:

```
## grug review

grug see nothing to review. no changes found. grug go back to cave
```

And stop.

**Filter out non-reviewable files:**
- Skip lockfiles, compiled output, vendor directories, binary files
- Skip pure deletion diffs (grug generally approve of removing code)

### Stage 2: Select Reviewers

**Always select:** `grug-complexity`, `grug-abstraction`, `grug-expression`, `grug-locality`

**For each conditional persona**, read the diff and decide whether to activate. Use judgment, not mechanical keyword matching:

- **grug-testing**: Are test files meaningfully changed? (Not just a renamed import or moved fixture.)
- **grug-type-shaman**: Are type definitions, generics, or complex type annotations being added or modified? (Not just a simple type annotation on a parameter.)
- **grug-concurrency**: Are concurrency primitives being introduced or meaningfully changed? (Not just calling an existing async function.)
- **grug-dry**: Are new helpers, utilities, or abstractions being introduced? Or does the diff show similar code patterns that might trigger DRY instincts?

**Announce the team** before spawning:

```
**grug review team:**
- grug-complexity (always-on)
- grug-abstraction (always-on)
- grug-expression (always-on)
- grug-locality (always-on)
- grug-testing: diff modifies test assertions and adds new test file
- grug-type-shaman: diff introduces generic utility types
```

Include one-line justifications only for conditional reviewers.

### Stage 3: Spawn Sub-Agents

For each selected reviewer, spawn a sub-agent using the Agent tool. **Spawn all reviewers in parallel** — make all Agent calls in a single message.

Each agent receives a prompt built from the subagent template with these variables filled:

| Variable | Source |
|----------|--------|
| `{persona_file}` | Content of `agents/grug/{persona-name}.md` |
| `{voice_guide}` | Content of `references/grug-voice-guide.md` |
| `{diff_scope_rules}` | Content of `references/diff-scope.md` |
| `{schema}` | Content of `references/findings-schema.json` |
| `{file_list}` | `FILES:` from Stage 1 |
| `{diff}` | `DIFF:` from Stage 1 |

**Agent spawn parameters:**
- `subagent_type`: Use the appropriate grug reviewer agent type (e.g., `compound-engineering:review:grug-complexity`)
- If grug-specific agent types are not registered, use `general-purpose` with the full prompt template injected

**Fallback:** If parallel Agent spawning is not supported, spawn sequentially. Note the limitation in coverage section.

**Collect** the JSON response from each agent. If an agent fails or returns invalid JSON, note it and continue — never crash the whole review because one persona failed.

### Stage 4: Merge, Gate, and Present

#### 4.1 Validate

Check each reviewer's JSON response against the findings schema. Drop malformed findings. Record drop count.

#### 4.2 Confidence Gate

Suppress findings with confidence below 0.70. Record the count of suppressed findings.

#### 4.3 Deduplicate

Compute a fingerprint for each finding: `normalize(file) + line_bucket(line, ±3) + normalize(title)`

- `normalize(file)`: lowercase, strip leading `./`
- `line_bucket(line, ±3)`: round line number to nearest multiple of 3
- `normalize(title)`: lowercase, strip punctuation, collapse whitespace

On fingerprint collision (same issue flagged by multiple reviewers):
- Keep the highest severity
- Keep the highest confidence
- Combine `grug_say` from both reviewers (use the longer/better one, note both reviewers)
- Union the evidence arrays

#### 4.4 Partition

Separate findings into three groups:
1. **Negative findings** (positive: false, pre_existing: false) → main findings table
2. **Positive findings** (positive: true) → "grug like" section
3. **Pre-existing findings** (pre_existing: true) → "things grug notice in passing" section

#### 4.5 Sort

Within each group, sort by:
1. Severity descending: `complexity demon` > `very bad` > `bad`
2. Confidence descending
3. File path alphabetically
4. Line number ascending

#### 4.6 Render

Use the output template to render the final report:
1. Header with scope and reviewer team
2. Findings tables grouped by severity (omit empty groups)
3. "grug like" section (omit if no positives)
4. "things grug notice in passing" section (omit if no pre-existing)
5. Coverage section (suppressed count, quiet reviewers, failed reviewers)
6. Verdict — synthesize in grug voice based on findings

**Collect `things_grug_not_sure_about`** from all reviewers. If there are notable uncertainties, append them to the coverage section:
```
- **grug not sure about:** {list of uncertainties from all reviewers}
```

#### 4.7 Quality Gate

Before outputting, verify:
- No findings below 0.70 confidence leaked through
- All `grug_say` fields follow the voice guide (third person, lowercase, short sentences)
- Evidence arrays are non-empty for every finding
- Severity labels are valid (`bad`, `very bad`, `complexity demon`)
- Verdict tone matches the severity distribution

Output the rendered report.
