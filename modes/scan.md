# Mode: scan — Portal Scanner (Offer Discovery)

Scans configured job portals, filters by title relevance, and adds new postings to the pipeline for later evaluation.

> **Note (v1.5+):** The default scanner (`scan.mjs` / `npm run scan`) is **zero-token** and queries the public Greenhouse, Ashby and Lever APIs directly. The Playwright/WebSearch tiers described below are the **agent** flow (run by Claude/Codex), not what `scan.mjs` does. If a company has no Greenhouse/Ashby/Lever API, `scan.mjs` will skip it; for those, the agent should cover them manually via Tier 1 (Playwright) or Tier 3 (WebSearch).

## Recommended execution

Run as a sub-agent so the main context isn't burned:

```
Agent(
    subagent_type="general-purpose",
    prompt="[contents of this file + specific data]",
    run_in_background=True
)
```

## Configuration

Read `portals.yml`, which contains:
- `search_queries`: List of WebSearch queries with `site:` filters per portal (broad discovery)
- `tracked_companies`: Specific companies with a `careers_url` for direct navigation
- `title_filter`: Positive/negative/seniority_boost keywords for title filtering

## Discovery strategy (3 tiers)

### Tier 1 — Playwright direct (PRIMARY)

**For each company in `tracked_companies`:** Navigate to its `careers_url` with Playwright (`browser_navigate` + `browser_snapshot`), read every visible job listing, and extract title + URL for each. This is the most reliable method because:
- It sees the page live (not Google's cached results)
- It works with SPAs (Ashby, Lever, Workday)
- It catches new postings instantly
- It doesn't depend on Google's indexing

**Every company MUST have a `careers_url` in portals.yml.** If it doesn't, look it up once, save it, and reuse it on future scans.

### Tier 2 — ATS APIs / feeds (COMPLEMENTARY)

For companies with a public API or structured feed, use the JSON/XML response as a quick complement to Tier 1. It's faster than Playwright and reduces visual-scraping errors.

**Current support (variables between `{}`):**
- **Greenhouse**: `https://boards-api.greenhouse.io/v1/boards/{company}/jobs`
- **Ashby**: `https://jobs.ashbyhq.com/api/non-user-graphql?op=ApiJobBoardWithTeams`
- **BambooHR**: list `https://{company}.bamboohr.com/careers/list`; posting detail `https://{company}.bamboohr.com/careers/{id}/detail`
- **Lever**: `https://api.lever.co/v0/postings/{company}?mode=json`
- **Teamtailor**: `https://{company}.teamtailor.com/jobs.rss`
- **Workday**: `https://{company}.{shard}.myworkdayjobs.com/wday/cxs/{company}/{site}/jobs`

**Parsing convention per provider:**
- `greenhouse`: `jobs[]` → `title`, `absolute_url`
- `ashby`: GraphQL `ApiJobBoardWithTeams` with `organizationHostedJobsPageName={company}` → `jobBoard.jobPostings[]` (`title`, `id`; build the public URL if it isn't in the payload)
- `bamboohr`: list `result[]` → `jobOpeningName`, `id`; build the detail URL `https://{company}.bamboohr.com/careers/{id}/detail`; to read the full JD, GET the detail and use `result.jobOpening` (`jobOpeningName`, `description`, `datePosted`, `minimumExperience`, `compensation`, `jobOpeningShareUrl`)
- `lever`: root array `[]` → `text`, `hostedUrl` (fallback: `applyUrl`)
- `teamtailor`: RSS items → `title`, `link`
- `workday`: `jobPostings[]`/`jobPostings` (depending on the tenant) → `title`, `externalPath` or URL built from the host

### Tier 3 — WebSearch queries (BROAD DISCOVERY)

The `search_queries` with `site:` filters cover portals cross-sectionally (every Ashby, every Greenhouse, etc.). Useful for finding NEW companies not yet in `tracked_companies`, but results can be stale.

**Execution priority:**
1. Tier 1: Playwright → every `tracked_companies` entry with a `careers_url`
2. Tier 2: API → every `tracked_companies` entry with `api:`
3. Tier 3: WebSearch → every `search_queries` entry with `enabled: true`

The tiers are additive — run them all, then merge and deduplicate the results.

## Workflow

1. **Read the config**: `portals.yml`
2. **Read history**: `data/scan-history.tsv` → URLs already seen
3. **Read dedup sources**: `data/applications.md` + `data/pipeline.md`

4. **Tier 1 — Playwright scan** (parallel in batches of 3-5):
   For each company in `tracked_companies` with `enabled: true` and a `careers_url`:
   a. `browser_navigate` to the `careers_url`
   b. `browser_snapshot` to read every job listing
   c. If the page has filters/departments, navigate the relevant sections
   d. For each job listing extract: `{title, url, company}`
   e. If the page paginates results, navigate the extra pages
   f. Accumulate into the candidate list
   g. If the `careers_url` fails (404, redirect), fall back to `scan_query` and flag the URL for update

5. **Tier 2 — ATS APIs / feeds** (parallel):
   For each company in `tracked_companies` with `api:` defined and `enabled: true`:
   a. WebFetch the API/feed URL
   b. If `api_provider` is defined, use its parser; if not, infer from the domain (`boards-api.greenhouse.io`, `jobs.ashbyhq.com`, `api.lever.co`, `*.bamboohr.com`, `*.teamtailor.com`, `*.myworkdayjobs.com`)
   c. For **Ashby**, POST with:
      - `operationName: ApiJobBoardWithTeams`
      - `variables.organizationHostedJobsPageName: {company}`
      - GraphQL query over `jobBoardWithTeams` + `jobPostings { id title locationName employmentType compensationTierSummary }`
   d. For **BambooHR**, the list carries only basic metadata. For each relevant item, read the `id`, GET `https://{company}.bamboohr.com/careers/{id}/detail`, and extract the full JD from `result.jobOpening`. Use `jobOpeningShareUrl` as the public URL if present; otherwise use the detail URL.
   e. For **Workday**, POST JSON with at least `{"appliedFacets":{},"limit":20,"offset":0,"searchText":""}` and paginate by `offset` until there are no more results
   f. For each job extract and normalise: `{title, url, company}`
   g. Accumulate into the candidate list (dedup with Tier 1)

6. **Tier 3 — WebSearch queries** (parallel where possible):
   For each query in `search_queries` with `enabled: true`:
   a. Run WebSearch with the defined `query`
   b. From each result extract: `{title, url, company}`
      - **title**: from the result title (before " @ " or " | ")
      - **url**: the result URL
      - **company**: after " @ " in the title, or extract from the domain/path
   c. Accumulate into the candidate list (dedup with Tiers 1+2)

6. **Filter by title** using `title_filter` from `portals.yml`:
   - At least 1 `positive` keyword must appear in the title (case-insensitive)
   - 0 `negative` keywords may appear
   - `seniority_boost` keywords raise priority but aren't required

7. **Deduplicate** against 3 sources:
   - `scan-history.tsv` → exact URL already seen
   - `applications.md` → company + normalised role already evaluated
   - `pipeline.md` → exact URL already pending or processed

7.5. **Verify liveness for Tier 3 (WebSearch) results** — BEFORE adding to the pipeline:

   WebSearch results can be stale (Google caches results for weeks or months). To avoid evaluating expired postings, verify every new Tier 3 URL with Playwright. Tiers 1 and 2 are inherently live and don't need this check.

   For every new Tier 3 URL (sequential — NEVER run Playwright in parallel):
   a. `browser_navigate` to the URL
   b. `browser_snapshot` to read the content
   c. Classify:
      - **Live**: role title visible + role description + visible Apply/Submit control inside the main content. Don't count generic header/navbar/footer text.
      - **Expired** (any of these signals):
        - Final URL contains `?error=true` (Greenhouse redirects this way when a posting is closed)
        - Page contains: "job no longer available" / "no longer open" / "position has been filled" / "this job has expired" / "page not found"
        - Only navbar and footer visible, no JD content (content < ~300 chars)
   d. If expired: log to `scan-history.tsv` with status `skipped_expired` and drop it
   e. If live: continue to step 8

   **Don't abort the whole scan if one URL fails.** If `browser_navigate` errors (timeout, 403, etc.), mark it `skipped_expired` and move on.

8. **For each new, verified posting that passes the filters**:
   a. Add to `pipeline.md` under "Pending": `- [ ] {url} | {company} | {title}`
   b. Log to `scan-history.tsv`: `{url}\t{date}\t{query_name}\t{title}\t{company}\tadded`

9. **Postings filtered out by title**: log to `scan-history.tsv` with status `skipped_title`
10. **Duplicate postings**: log with status `skipped_dup`
11. **Expired postings (Tier 3)**: log with status `skipped_expired`

## Extracting title and company from WebSearch results

WebSearch results come in formats like: `"Job Title @ Company"` or `"Job Title | Company"` or `"Job Title — Company"`.

Extraction patterns per portal:
- **Ashby**: `"Senior AI PM (Remote) @ EverAI"` → title: `Senior AI PM`, company: `EverAI`
- **Greenhouse**: `"AI Engineer at Anthropic"` → title: `AI Engineer`, company: `Anthropic`
- **Lever**: `"Product Manager - AI @ Temporal"` → title: `Product Manager - AI`, company: `Temporal`

Generic regex: `(.+?)(?:\s*[@|—–-]\s*|\s+at\s+)(.+?)$`

## Private URLs

If a URL isn't publicly accessible:
1. Save the JD to `jds/{company}-{role-slug}.md`
2. Add to pipeline.md as: `- [ ] local:jds/{company}-{role-slug}.md | {company} | {title}`

## Scan History

`data/scan-history.tsv` tracks EVERY URL seen:

```
url	first_seen	portal	title	company	status
https://...	2026-02-10	Ashby — AI PM	PM AI	Acme	added
https://...	2026-02-10	Greenhouse — SA	Junior Dev	BigCo	skipped_title
https://...	2026-02-10	Ashby — AI PM	SA AI	OldCo	skipped_dup
https://...	2026-02-10	WebSearch — AI PM	PM AI	ClosedCo	skipped_expired
```

## Output summary

```
Portal Scan — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━
Queries run: N
Postings found: N total
Filtered by title: N relevant
Duplicates: N (already evaluated or in pipeline)
Expired drops: N (dead links, Tier 3)
New items added to pipeline.md: N

  + {company} | {title} | {query_name}
  ...

→ Run /career-ops pipeline to evaluate the new postings.
```

## careers_url maintenance

Every company in `tracked_companies` should have a `careers_url` — the direct link to its jobs page. This avoids re-discovering it on every scan.

**RULE: Always prefer the company's own careers page; fall back to the ATS endpoint only if the company has no proper corporate page.**

`careers_url` should point to the company's own careers page whenever one exists. Many companies run on Workday, Greenhouse or Lever under the hood but only expose the job IDs through their corporate domain. Using the raw ATS URL when a corporate page exists can cause false 410 errors because the job IDs don't match.

| ✅ Correct (corporate) | ❌ Avoid as first choice (direct ATS) |
|---|---|
| `https://careers.mastercard.com` | `https://mastercard.wd1.myworkdayjobs.com` |
| `https://openai.com/careers` | `https://job-boards.greenhouse.io/openai` |
| `https://stripe.com/jobs` | `https://jobs.lever.co/stripe` |

Fallback: if you only have the direct ATS URL, first go to the company website and locate its corporate careers page. Use the direct ATS URL only when the company has no corporate page of its own.

**Known patterns per platform:**
- **Ashby:** `https://jobs.ashbyhq.com/{slug}`
- **Greenhouse:** `https://job-boards.greenhouse.io/{slug}` or `https://job-boards.eu.greenhouse.io/{slug}`
- **Lever:** `https://jobs.lever.co/{slug}`
- **BambooHR:** list `https://{company}.bamboohr.com/careers/list`; detail `https://{company}.bamboohr.com/careers/{id}/detail`
- **Teamtailor:** `https://{company}.teamtailor.com/jobs`
- **Workday:** `https://{company}.{shard}.myworkdayjobs.com/{site}`
- **Custom:** The company's own URL (e.g., `https://openai.com/careers`)

**API/feed patterns per platform:**
- **Ashby API:** `https://jobs.ashbyhq.com/api/non-user-graphql?op=ApiJobBoardWithTeams`
- **BambooHR API:** list `https://{company}.bamboohr.com/careers/list`; detail `https://{company}.bamboohr.com/careers/{id}/detail` (`result.jobOpening`)
- **Lever API:** `https://api.lever.co/v0/postings/{company}?mode=json`
- **Teamtailor RSS:** `https://{company}.teamtailor.com/jobs.rss`
- **Workday API:** `https://{company}.{shard}.myworkdayjobs.com/wday/cxs/{company}/{site}/jobs`

**If `careers_url` is missing** for a company:
1. Try the known platform pattern
2. If that fails, run a quick WebSearch: `"{company}" careers jobs`
3. Navigate with Playwright to confirm it works
4. **Save the URL in portals.yml** for future scans

**If `careers_url` returns 404 or a redirect:**
1. Note it in the output summary
2. Try `scan_query` as a fallback
3. Flag for manual update

## portals.yml maintenance

- **ALWAYS save `careers_url`** when adding a new company
- Add new queries as portals or interesting roles surface
- Disable queries with `enabled: false` if they generate too much noise
- Adjust filter keywords as target roles evolve
- Add companies to `tracked_companies` when it's worth tracking them closely
- Verify `careers_url` periodically — companies swap ATS platforms
