# Mode: auto-pipeline — Full Automatic Pipeline

When the user pastes a JD (text or URL) without an explicit sub-command, run the WHOLE pipeline end-to-end:

## Step 0 — Extract the JD

If the input is a **URL** (not JD text pasted in), follow this strategy to pull the content:

**Priority order:**

1. **Playwright (preferred):** Most job portals (Lever, Ashby, Greenhouse, Workday) are SPAs. Use `browser_navigate` + `browser_snapshot` to render and read the JD.
2. **WebFetch (fallback):** For static pages (ZipRecruiter, WeLoveProduct, company career pages).
3. **WebSearch (last resort):** Search for role title + company on secondary portals that index the JD in static HTML.

**If nothing works:** Ask the candidate to paste the JD manually or share a screenshot.

**If the input is JD text** (not a URL): use it directly, no fetch needed.

## Step 1 — A-G evaluation
Run exactly as in the `oferta` mode (read `modes/oferta.md` for every block A-F + Block G Posting Legitimacy).

## Step 2 — Save the report .md
Save the full evaluation to `reports/{###}-{company-slug}-{YYYY-MM-DD}.md` (see format in `modes/oferta.md`).
Include Block G in the saved report. Add `**Legitimacy:** {tier}` to the report header.

## Step 3 — Generate PDF
Run the full `pdf` pipeline (read `modes/pdf.md`).

## Step 4 — Draft application answers (only if score >= 4.5)

If the final score is >= 4.5, draft answers for the application form:

1. **Extract the form's questions**: Use Playwright to navigate to the form and snapshot it. If they can't be extracted, use the generic questions.
2. **Generate the answers** in the tone described below.
3. **Save in the report** as `## H) Draft Application Answers`.

### Generic questions (use if the form's own questions can't be extracted)

- Why are you interested in this role?
- Why do you want to work at [Company]?
- Tell us about a relevant project or achievement
- What makes you a good fit for this position?
- How did you hear about this role?

### Tone for form answers

**Stance: "I'm choosing you."** The candidate has options and is picking this company for concrete reasons.

**Tone rules:**
- **Confident without arrogance**: "I've spent the past year building production AI agent systems — your role is where I want to apply that experience next"
- **Selective without snobbery**: "I've been intentional about finding a team where I can contribute meaningfully from day one"
- **Specific and concrete**: Always reference something REAL from the JD or the company, and something REAL from the candidate's experience
- **Direct, no fluff**: 2-4 sentences per answer. No "I'm passionate about..." or "I would love the opportunity to..."
- **The hook is the proof, not the claim**: Instead of "I'm great at X", say "I built X that does Y"

**Framework per question:**
- **Why this role?** → "Your [specific thing] maps directly to [specific thing I built]."
- **Why this company?** → Reference something concrete about the company. "I've been using [product] for [time/purpose]."
- **Relevant experience?** → A quantified proof point. "Built [X] that [metric]. Sold the company in 2025."
- **Good fit?** → "I sit at the intersection of [A] and [B], which is exactly where this role lives."
- **How did you hear?** → Honest: "Found through [portal/scan], evaluated against my criteria, and it scored highest."

**Language**: Always in the language of the JD (EN default). Apply `/tech-translate`.

## Step 5 — Update the tracker
Log in `data/applications.md` with every column, including Report and PDF marked ✅.

**If any step fails**, keep going with the remaining steps and mark the failed step as pending in the tracker.
