# Board Subagent: instructions

You search ONE job board for open job postings and return structured data. You do NOT judge fit to any CV; the main agent does that.

You receive: `board`, `queries` (list), `exclude` (terms), `date`, `max_age_days`, `unknown_date` (`exclude` | `include`).

## Freshness rule (strict)
Only return postings published within the last `max_age_days` days before `date`.
- Work out each posting's **posted date** as an ISO date (`YYYY-MM-DD`). Sources, in order of preference: the date on the posting page; relative text ("posted 3 days ago", "לפני 3 ימים", "פורסם היום") converted using `date`; ATS data (Greenhouse API `updated_at`/`first_published`, Workable/Lever posting date); the date shown in the search snippet; the date on the original board page when the text came from a mirror.
- A LinkedIn job ID or an old board ID is **not** proof of a date; use it only to rule a posting out (clearly old).
- Older than `max_age_days` → skip it (don't fetch further).
- No date can be found → keep it only if `unknown_date` is `include`, with `"posted": "unknown"`. Otherwise skip it.
- Search tip: add freshness words to queries where the site supports them (e.g. "היום", "השבוע", "past week") and prefer board search pages sorted by newest.
Project folder: the repo root (`job_seeker`), the same working folder as the lead agent. All paths are relative to it.

## Steps
1. Read `seen_jobs.json` and skip any URL already listed.
2. Run each query with WebSearch. Add your own 1–2 variations if results are thin.
3. Keep only links to a **single posting** (not search/list pages, articles, courses or profiles).
4. Fetch each posting's full text with WebFetch. If a fetch fails (403/410/redirect), use the fallbacks below. If none work, keep the item with `fetch_ok: false` and whatever the search snippet showed.
5. Skip postings that are closed ("no longer accepting applications", 410 with no mirror), that fail the freshness rule above, or that match an `exclude` term in the title/requirements.
6. Aim for up to 8 good candidates. Quality over quantity.
7. Write the results to `runs/<date>/<board>.json` and return the same JSON as your final answer.

## Per-board notes
- **linkedin**: `site:linkedin.com/jobs/view` returns almost nothing new, so search `"<terms>" Israel linkedin jobs` and use the jobify360.co.il `-in` mirror copies of LinkedIn ads for the full text (also builtin.com, smartrecruiters). aijobs.net no longer works (redirects to foorilla.com). A LinkedIn job ID below about 4400000000 usually means an older posting, so check its age.
- **alljobs**: `site:alljobs.co.il` search works. Direct fetches of `alljobs.co.il/m/p/jobs/<id>` always return 410, so fetch the jobify360.co.il mirror (`/jobs/<id>-aj` or `-vip`). The mirror has no posted date. Hebrew queries work best.
- **drushim**: `site:drushim.co.il` search works. Most direct fetches return 410, so fetch the jobify360.co.il `-dr` mirror. Remove duplicate job IDs (the same ad is often reposted).
- **jobmaster**: `site:` search returns nothing for jobmaster. Use JobMaster's own search page with the Hebrew/English terms instead (only page 1 is readable; later pages need a login). Postings are at `jobmaster.co.il/jobs/checknum.asp?key=<id>` and show "X days ago".
- **career_pages**: company career sites and ATS/VC job boards: jobs.workable.com, Greenhouse (use the public API `boards-api.greenhouse.io/v1/boards/<company>/jobs` when board pages only show lists), jobs.lever.co, smartrecruiters, jobs.tlv.partners, careers.grovevc.com, jobs.detroit.vc, company `/careers` pages. `site:comeet.com` returned nothing useful. Add `Israel` to queries.

## Output (JSON array)
```json
[
  {
    "board": "alljobs",
    "url": "https://...",
    "apply_url": "https://...",
    "apply_via": "AllJobs",
    "title": "...",
    "company": "...",
    "location": "...",
    "posted": "YYYY-MM-DD or unknown",
    "posted_source": "page | relative text | ATS | snippet | original board",
    "open": true,
    "fetch_ok": true,
    "must_haves": ["verbatim or close paraphrase", "..."],
    "nice_to_haves": ["..."],
    "years_required": "e.g. 3+ or unknown",
    "degree": "... or none stated",
    "responsibilities": "2–3 line summary",
    "language": "he|en"
  }
]
```
`apply_url` + `apply_via` are **REQUIRED for every item**. They are the page where the user actually submits the application, never a mirror:
| Source | apply_url | apply_via |
|---|---|---|
| AllJobs (incl. jobify360 `-aj`/`-vip`) | `https://www.alljobs.co.il/Search/UploadSingle.aspx?JobID=<id>` | `AllJobs` |
| Drushim (incl. jobify360 `-dr`) | the original `https://www.drushim.co.il/job/<id>/<hash>/` URL (find it via search if only the mirror is known) | `Drushim` |
| JobMaster | `https://www.jobmaster.co.il/jobs/checknum.asp?key=<id>` | `JobMaster` |
| LinkedIn (incl. jobify360 `-in`) | `https://www.linkedin.com/jobs/view/<id>`. If the LinkedIn page shows an external "Apply" link, use that company link instead | `LinkedIn` or the company/ATS name |
| Workable | `https://jobs.workable.com/view/<id>` (the "Apply for this job" target) | `Workable` |
| Greenhouse / Lever / Comeet / SmartRecruiters / company careers | the posting's apply button target (e.g. Greenhouse `.../jobs/<id>#app`, Lever `.../<id>/apply`) | ATS or company name |
jobify360 pages are mirrors: their "Upload CV" applies through jobify360, not to the employer, so never use them as `apply_url`.

**Closed check:** also mark `open: false` if any copy of the ad says it was removed, or if the posting names an application deadline / hiring-event date that has already passed.

Translate Hebrew requirements into English in `must_haves` / `nice_to_haves` (keep the Hebrew title in `title`).
