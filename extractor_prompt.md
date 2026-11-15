# ATS Keyword Extractor

One extractor, used the same way for every input document (a job description OR a CV).
Run it only when the user asks (new/replaced JD, or updated CV). The daily agent never runs it.

## Input
A single document: a job description (`job_description.txt` or a new JD file) or the CV (`cv.md`).

## Output
A profile file in `profiles/`:
- JD → `profiles/jd_profile.md`
- CV → `profiles/cv_profile.md`

Each profile uses exactly these sections, in this order, so profiles can be compared term by term:

```
# <JD|CV> Profile
Source: <file name> · Extracted: <YYYY-MM-DD>

## Skills          (competencies / methods: e.g. prompt engineering, process mapping, systems thinking)
## Tools           (named technologies / products / languages: e.g. Python, SQL, MCP, n8n)
## Responsibilities (action phrases an ATS would match: e.g. requirements gathering, end-to-end implementation)
## Domain          (industry / subject areas: e.g. AI agents, data quality, security)
## Seniority       (years, level, degree)
## Titles          (job titles stated or clearly implied, incl. Hebrew equivalents)
```

## Extraction rules
1. Use **ATS-style terms**: short, canonical noun phrases recruiters and ATS filters match on (2–4 words max). Prefer the industry-standard spelling ("REST API", "LLM", "prompt engineering").
2. **Normalize** to a canonical term and add common variants in parentheses, e.g. `API integrations (integrations, REST API integration)`.
3. Extract what the document **states or directly demonstrates**. No guessing. For a CV, an achievement counts as evidence for the skill it shows (e.g. "transformed manual tasks into agentic workflows" → agentic workflows, automation, process improvement).
4. Each term appears once, in its single best section.
5. Never include personal identifiers (name, email, phone, URLs) in a profile.

## Merge → `keywords.md`
After one or both profiles are (re)extracted, merge them into `keywords.md`:
- **Core ATS** = terms in BOTH profiles (match on canonical term or variants).
- **JD extras** = terms only in the JD profile.
- **CV strengths** = terms only in the CV profile.
- **Titles** = union of both Titles sections.
- **Exclude** and **Filters** come from Seniority + user rules.

Preserve user edits on every re-merge:
- Everything under `## My additions` is never touched.
- A term disabled with a leading `~` stays disabled.
- A term the user deleted is listed in `## Removed` and must not be re-added.
