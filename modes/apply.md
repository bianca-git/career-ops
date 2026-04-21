# Mode: apply — Live Application Assistant

Interactive mode for when the candidate is filling in an application form in Chrome. Reads what's on screen, loads prior context for the posting, and drafts tailored answers to every question on the form.

## Requirements

- **Best with Playwright visible**: In visible mode the candidate sees the browser and Claude can interact with the page.
- **Without Playwright**: the candidate shares a screenshot or pastes the questions manually.

## Workflow

```
1. DETECT     → Read the active Chrome tab (screenshot/URL/title)
2. IDENTIFY   → Extract company + role from the page
3. LOOK UP    → Match against existing reports in reports/
4. LOAD       → Read the full report + Section G (if present)
5. COMPARE    → Does the role on screen match the one evaluated? If it changed → flag it
6. ANALYSE    → Identify EVERY visible question on the form
7. GENERATE   → For each question, produce a tailored answer
8. PRESENT    → Show the answers formatted for copy-paste
```

## Step 1 — Detect the posting

**With Playwright:** Snapshot the active page. Read title, URL, and visible content.

**Without Playwright:** Ask the candidate to:
- Share a screenshot of the form (the Read tool accepts images)
- Or paste the form questions as text
- Or give the company + role so we can look it up

## Step 2 — Identify and load context

1. Pull the company name and role title from the page
2. Search `reports/` by company name (Grep, case-insensitive)
3. If there's a match → load the full report
4. If Section G is present → use its prior draft answers as a baseline
5. If no match → flag it and offer to run a quick auto-pipeline

## Step 3 — Detect role changes

If the role on screen differs from the evaluated one:
- **Flag to the candidate**: "The role has changed from [X] to [Y]. Want me to re-evaluate, or adapt the answers to the new title?"
- **If adapting**: Adjust the answers to the new role without re-evaluating
- **If re-evaluating**: Run the full A-F evaluation, update the report, regenerate Section G
- **Update the tracker**: Change the role title in applications.md if needed

## Step 4 — Parse the form

Identify EVERY visible question:
- Free-text fields (cover letter, why this role, etc.)
- Dropdowns (how did you hear, work authorisation, etc.)
- Yes/No (relocation, visa, etc.)
- Salary fields (range, expectation)
- Upload fields (resume, cover letter PDF)

Classify each question:
- **Already answered in Section G** → adapt the existing answer
- **New question** → generate an answer from the report + cv.md

## Step 5 — Generate the answers

For each question, draft the answer using:

1. **Report context**: Use proof points from Block B and STAR stories from Block F
2. **Prior Section G**: If a draft answer exists, use it as a base and refine
3. **"I'm choosing you" tone**: Same framework as the auto-pipeline
4. **Specificity**: Reference something concrete from the JD visible on screen
5. **career-ops proof point**: Include in "Additional info" if there's a field for it

**Output format:**

```
## Answers for [Company] — [Role]

Based on: Report #NNN | Score: X.X/5 | Archetype: [type]

---

### 1. [Exact question from the form]
> [Answer ready for copy-paste]

### 2. [Next question]
> [Answer]

...

---

Notes:
- [Any observations about the role, changes, etc.]
- [Personalisation suggestions the candidate should review]
```

## Step 6 — Post-apply (optional)

If the candidate confirms they submitted the application:
1. Update the status in `applications.md` from `Evaluated` to `Applied`
2. Update Section G of the report with the final answers
3. Suggest the next step: `/career-ops contacto` for LinkedIn outreach

## Scroll handling

If the form has more questions than are visible:
- Ask the candidate to scroll and share another screenshot
- Or paste the remaining questions
- Process in iterations until every question on the form is covered
