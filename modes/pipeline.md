# Mode: pipeline — URL Inbox (Second Brain)

Processes URLs accumulated in `data/pipeline.md`. The user drops URLs in whenever they like, then runs `/career-ops pipeline` to process them all at once.

## Workflow

1. **Read** `data/pipeline.md` → find `- [ ]` items under the "Pending" section
2. **For each pending URL**:
   a. Work out the next sequential `REPORT_NUM` (read `reports/`, take max + 1)
   b. **Extract the JD** using Playwright (browser_navigate + browser_snapshot) → WebFetch → WebSearch
   c. If the URL isn't reachable → mark it as `- [!]` with a note and keep going
   d. **Run the full auto-pipeline**: A-F evaluation → Report .md → PDF (if score >= 3.0) → Tracker
   e. **Move from "Pending" to "Processed"**: `- [x] #NNN | URL | Company | Role | Score/5 | PDF ✅/❌`
3. **If there are 3+ pending URLs**, launch agents in parallel (Agent tool with `run_in_background`) to maximise speed.
4. **When done**, show a summary table:

```
| # | Company | Role | Score | PDF | Recommended action |
```

## pipeline.md format

```markdown
## Pending
- [ ] https://jobs.example.com/posting/123
- [ ] https://boards.greenhouse.io/company/jobs/456 | Company Inc | Senior PM
- [!] https://private.url/job — Error: login required

## Processed
- [x] #143 | https://jobs.example.com/posting/789 | Acme Corp | AI PM | 4.2/5 | PDF ✅
- [x] #144 | https://boards.greenhouse.io/xyz/jobs/012 | BigCo | SA | 2.1/5 | PDF ❌
```

## Smart JD detection from a URL

1. **Playwright (preferred):** `browser_navigate` + `browser_snapshot`. Works with every SPA.
2. **WebFetch (fallback):** For static pages or when Playwright isn't available.
3. **WebSearch (last resort):** Search on secondary portals that index the JD.

**Special cases:**
- **LinkedIn**: May require login → mark `[!]` and ask the user to paste the text
- **PDF**: If the URL points at a PDF, read it directly with the Read tool
- **`local:` prefix**: Read the local file. Example: `local:jds/linkedin-pm-ai.md` → read `jds/linkedin-pm-ai.md`

## Automatic numbering

1. List every file in `reports/`
2. Extract the number prefix (e.g., `142-medispend...` → 142)
3. New number = highest found + 1

## Source sync

Before processing any URL, check sync:
```bash
node cv-sync-check.mjs
```
If things are out of sync, warn the user before continuing.
