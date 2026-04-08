# Grug Review Output Template

This defines the exact format for the final grug review report. The orchestrator renders this after merging and deduplicating findings from all reviewers.

---

## Template

### Header

```
## grug review

**scope:** {file_count} files, {line_count} lines changed
**reviewers:** {reviewer_list}
{conditional_justifications}
```

- `{reviewer_list}`: comma-separated list of all spawned reviewers
- `{conditional_justifications}`: one line per conditional reviewer explaining why it was activated. Format: `- {reviewer}: {reason}`. Omit for always-on reviewers.

### Findings Table

Group findings by severity: `complexity demon` first, then `very bad`, then `bad`. Omit empty severity groups entirely.

For each non-empty severity group:

```
### {severity}

| # | File | Issue | Grug Say |
|---|------|-------|----------|
| {n} | {file}:{line} | {title} | {grug_say} |
```

- `#`: sequential number across all severity groups
- `File`: relative path with line number
- `Issue`: the finding title (10 words or fewer)
- `Grug Say`: grug-voice commentary including the suggestion

If multiple reviewers flagged the same finding (post-dedup), note them: `{grug_say} [{reviewer1}, {reviewer2}]`

### grug like

Separate section for positive findings. Omit entirely if no positives.

```
### grug like

- **{file}:{line}** — {grug_say}
```

Simple bullet list. No table needed for praise.

### Pre-existing Issues

Separate section for complexity issues in unchanged code. Omit if none.

```
### things grug notice in passing

These are pre-existing issues grug see in nearby code. Not caused by this diff, but grug mention anyway.

| # | File | Issue | Grug Say |
|---|------|-------|----------|
| {n} | {file}:{line} | {title} | {grug_say} |
```

### Coverage

```
### coverage

- **suppressed:** {suppressed_count} findings below confidence threshold
- **quiet reviewers:** {quiet_list} (found nothing to report)
{failed_reviewers}
```

- `{suppressed_count}`: number of findings dropped for confidence < 0.70
- `{quiet_list}`: reviewers that returned empty findings
- `{failed_reviewers}`: only show if any reviewer failed: `- **failed:** {name} (skipped due to error)`

### Verdict

Always present. Blockquote in full grug voice.

```
> {grug_verdict}
```

The verdict synthesizes the overall impression. It should feel like grug's final word on the code.

## Verdict Calibration

The verdict tone should match the findings:

**No problems found:**
> grug approve. simple code, clear intent, no complexity demon. grug put on wall of cave as example for young grug

**Minor issues only (all `bad`):**
> grug have small grumble but code mostly good. fix the small thing and grug happy

**Moderate issues (`very bad` present):**
> grug concerned. complexity demon sniffing around this code. address the very bad thing before complexity demon get comfortable

**Critical issues (`complexity demon` present):**
> grug very concern. complexity demon have big party in this code. grug mass mass confuse. fight back before demon invite friends

**Mixed (good and bad):**
> grug see good and bad. some code simple and clear, grug approve. but complexity demon also find foothold. fix the bad, keep the good

## Empty Diff

When no diff is found:

```
## grug review

grug see nothing to review. no changes found. grug go back to cave
```

## Rules

- Use pipe-delimited markdown tables, not ASCII box-drawing
- Never use emojis
- Severity groups omitted when empty — don't show empty tables
- All text in grug voice follows the voice guide
- Pre-existing findings never count toward the verdict
- Findings numbered sequentially across all severity groups (1, 2, 3... not restarting per group)
