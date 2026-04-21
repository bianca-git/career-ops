# Mode: batch — Bulk Posting Processing

Two ways to use it: **conductor --chrome** (navigates portals live) or **standalone** (script for already-collected URLs).

## Architecture

```
Claude Conductor (claude --chrome --dangerously-skip-permissions)
  │
  │  Chrome: navigates portals (logged-in sessions)
  │  Reads DOM directly — the user sees everything live
  │
  ├─ Posting 1: reads JD from the DOM + URL
  │    └─► claude -p worker → report .md + PDF + tracker-line
  │
  ├─ Posting 2: clicks next, reads JD + URL
  │    └─► claude -p worker → report .md + PDF + tracker-line
  │
  └─ Done: merge tracker-additions → applications.md + summary
```

Each worker is a `claude -p` child with a fresh 200K-token context. The conductor only orchestrates.

## Files

```
batch/
  batch-input.tsv               # URLs (by conductor or manual)
  batch-state.tsv               # Progress (auto-generated, gitignored)
  batch-runner.sh               # Standalone orchestrator script
  batch-prompt.md               # Prompt template for workers
  logs/                         # One log per posting (gitignored)
  tracker-additions/            # Tracker lines (gitignored)
```

## Mode A: Conductor --chrome

1. **Read state**: `batch/batch-state.tsv` → know what's already processed
2. **Navigate the portal**: Chrome → search URL
3. **Extract URLs**: Read the results DOM → extract the list of URLs → append to `batch-input.tsv`
4. **For each pending URL**:
   a. Chrome: click into the posting → read JD text from the DOM
   b. Save the JD to `/tmp/batch-jd-{id}.txt`
   c. Work out the next sequential REPORT_NUM
   d. Run via Bash:
      ```bash
      claude -p --dangerously-skip-permissions \
        --append-system-prompt-file batch/batch-prompt.md \
        "Process this posting. URL: {url}. JD: /tmp/batch-jd-{id}.txt. Report: {num}. ID: {id}"
      ```
   e. Update `batch-state.tsv` (completed/failed + score + report_num)
   f. Log to `logs/{report_num}-{id}.log`
   g. Chrome: go back → next posting
5. **Pagination**: When there are no more postings → click "Next" → repeat
6. **Done**: Merge `tracker-additions/` → `applications.md` + summary

## Mode B: Standalone script

```bash
batch/batch-runner.sh [OPTIONS]
```

Options:
- `--dry-run` — list pending without running
- `--retry-failed` — only retry failures
- `--start-from N` — start from ID N
- `--parallel N` — N workers in parallel
- `--max-retries N` — attempts per posting (default: 2)

## batch-state.tsv format

```
id	url	status	started_at	completed_at	report_num	score	error	retries
1	https://...	completed	2026-...	2026-...	002	4.2	-	0
2	https://...	failed	2026-...	2026-...	-	-	Error msg	1
3	https://...	pending	-	-	-	-	-	0
```

## Resumability

- If it dies → re-run → reads `batch-state.tsv` → skips completed
- Lock file (`batch-runner.pid`) prevents double execution
- Each worker is independent: a failure on posting #47 doesn't affect the rest

## Workers (claude -p)

Each worker gets `batch-prompt.md` as its system prompt. It's self-contained.

The worker produces:
1. Report `.md` in `reports/`
2. PDF in `output/`
3. Tracker line in `batch/tracker-additions/{id}.tsv`
4. Result JSON on stdout

## Error handling

| Error | Recovery |
|-------|----------|
| URL unreachable | Worker fails → conductor marks `failed`, moves on |
| JD behind a login | Conductor tries to read the DOM. If that fails → `failed` |
| Portal changes layout | Conductor reasons over the HTML and adapts |
| Worker crashes | Conductor marks `failed`, moves on. Retry with `--retry-failed` |
| Conductor dies | Re-run → reads state → skips completed |
| PDF fails | Report .md is saved. PDF stays pending |
