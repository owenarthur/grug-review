# Diff Scope Rules

How grug reviewers decide what to review and how to classify findings.

## Scope Discovery

Determine what code to review, in priority order:

1. **User-specified scope** — if `BASE:`, `FILES:`, `DIFF:` markers are provided, use those directly
2. **Uncommitted changes** — `git diff HEAD` if the working tree has modifications
3. **Unpushed commits** — `git diff $(git merge-base origin/HEAD HEAD)..HEAD` for committed-but-unpushed work
4. **Branch argument** — if a branch name is provided, `git diff $(git merge-base HEAD <branch>)..<branch>`

## Finding Classification

Two tiers. Keep it simple.

### Primary

Lines added or modified in the diff. This is what grug reviews.

- Main review focus
- Report at full confidence
- These are the lines the developer just wrote or changed

### Pre-existing

Unchanged code unrelated to the diff.

- Mark `pre_existing: true`
- Reported in a separate section
- Does not count toward the verdict
- Only flag pre-existing issues when they are clearly problematic (complexity demon level)
- Do not go hunting through the whole file for pre-existing issues — grug reviews what changed

**The rule:** if the same issue would appear on a diff that only touched these lines, it's primary. If the code was already there and the diff didn't touch it or interact with it, it's pre-existing.

## What Not to Review

- Generated files (lockfiles, compiled output, vendor directories)
- Configuration files unless they introduce complexity (e.g., a 200-line webpack config)
- Pure deletion diffs (grug generally approve of removing code)
- Binary files
