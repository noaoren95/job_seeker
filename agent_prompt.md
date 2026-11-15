# Daily Job Agent: main agent (orchestrator) instructions

Project folder: `G:\My Drive\builder_projects\job_seeker`

Architecture:
```
main agent (this file)
  ├─ reads keywords.md, builds queries for each board
  ├─ spawns one board subagent per job board IN PARALLEL  (subagent_board_prompt.md)
  │     linkedin · alljobs · drushim · jobmaster · career_pages
  │     each: search → fetch postings → extract requirements → runs/<date>/<board>.json
  ├─ merges + dedupes candidates
  ├─ CV-fit gate + scoring (here only, so every job is judged the same way)
  └─ email + update seen_jobs.json / rejected_log.md
```

## Inputs (read fresh every run)
- `keywords.md`: the live keyword list. Ignore lines starting with `~` and everything under `## Removed`. Terms under `## My additions` count toward the group in their tag (e.g. `[core]`).
- `cv.md`: the user's full CV (git-ignored; see `cv.example.md` for the format), used ONLY here for the gate. It is never passed to subagents.
- `seen_jobs.json`: postings already handled (git-ignored; create it as `{"seen": []}` if missing).
- Do **not** read `job_description.txt` or run the extractor.

Privacy: no name, email, phone or LinkedIn URL from the CV goes into any query, URL or subagent prompt.

## 1. Build queries for each board
For each board, build 3–4 queries from enabled terms:
- Pair 2–3 **Core ATS** terms (including `[core]` additions), sometimes adding one **CV strength** or one **Title**. Never search a title alone.
- Rotate combinations from day to day (based on the date) so coverage grows.
- Hebrew boards (AllJobs, Drushim, JobMaster) get at least one Hebrew query (e.g. `AI אוטומציה אינטגרציות`, `מיישם AI`, `סוכני AI`).

## 2. Spawn board subagents (parallel, one message)
Read `Max job age (days)` and `Unknown posting date` from the **Filters** section of `keywords.md` (defaults if missing: 7 and `exclude`).
Spawn 5 subagents, one per board, in a single message so they run at the same time. Each prompt =
the contents of `subagent_board_prompt.md` + the board name + its queries + the Exclude terms + today's date + `max_age_days` + `unknown_date`.
Boards: `linkedin`, `alljobs`, `drushim`, `jobmaster`, `career_pages`.

## 3. Merge
- Read `runs/<date>/*.json`. Dedupe the same role across boards (same company + similar title); keep the URL with the most complete text and record all sources.
- Drop anything already in `seen_jobs.json`, closed, or not in Israel/remote.
- **Freshness gate (applied again here):** drop any posting whose `posted` date is more than `max_age_days` days before today. If `posted` is `unknown`, drop it when `unknown_date` is `exclude`; when it is `include`, keep it and show "date unknown" on its card. Record dropped postings as `status: "stale"` in `seen_jobs.json` only when their date is known (unknown-date ones may appear again with a date later).

## 4. CV-fit gate (hard, all must pass)
For each posting's **must-have** list:
1. **Evidence**: every must-have is backed by a concrete CV line. Write the CV line next to each one. Any must-have with no backing → REJECT.
   - Close equivalents count ("REST APIs" ← Postman/Swagger + API work; "LLMs" ← Claude/GPT/Gemini work; "automation" ← agentic workflows / n8n; "customer-facing" ← technical bridge with Customer Experience).
   - Don't stretch: a required language/platform the CV doesn't mention (e.g. TypeScript, Java, Kubernetes, Salesforce/ERP expertise) is not backed.
2. **Seniority**: work out the total years of experience from the dates in `cv.md`. REJECT if the posting needs clearly more (e.g. 8+ years) or a Staff/Principal/Director level, or if a specific required experience type exceeds what the CV shows (e.g. "5+ yrs software engineering" when the CV has no software-engineer role).
3. **Degree**: compare with the degree in `cv.md`. REJECT if the posting strictly requires a different degree field or a higher degree (MSc/PhD).
4. **Location**: Israel or remote-from-Israel.
5. **Role fit**: overlaps with AI solutions / agents / integrations / automation / operations / technical customer-facing. REJECT pure sales, research, or unrelated roles.

## 5. Score (only jobs that pass the gate), 0–10
- 4 pts: share of required + nice-to-have terms covered by the CV
- 3 pts: overlap with **Core ATS** terms
- 3 pts: closeness to the target direction (AI solutions engineering)
Recommend scores **≥ 7**.

## 6. Output
- Recommended jobs (best first): title, company, location, source(s), score, URL, CV evidence (2–3 bullets), matched keywords, gaps (nice-to-haves not in the CV).
- Add every evaluated posting to `seen_jobs.json` with `status`: `sent` | `rejected` | `low_score` | `closed`.
- Overwrite `rejected_log.md` with this run's rejected/low-score jobs: title, company, URL, one-line reason.
- Save a copy of the email HTML to `runs/<date>/email.html` and a summary to `runs/<date>/results.md`.

## 7. Email
Send with the Gmail connector `send_message` to the user's own address (the account the Gmail connector is signed in to), using `htmlBody` + a plain-text `body` fallback.
- **Subject:** `Daily job matches – <D Mon YYYY> (<N> new)`
- **Layout** (inline CSS only; Gmail strips `<style>`): follow `templates/email_template.html` as the reference template.
  1. Header bar: date, N new matches, postings checked, boards searched, number rejected, and the active freshness window ("jobs from the last <max_age_days> days").
  2. One card per recommended job, best first: score badge (green ≥ 8, blue 7–7.9) + linked title; company · location · source(s) · **posted date (always shown, e.g. "posted 2 days ago · 13 Nov")**; **CV evidence** (2–3 points); **Matched** keywords; **Gaps / Note** (orange); and a **REQUIRED** button at the bottom of each card: `Apply on <apply_via> →` linking to `apply_url`. Button style: `display:inline-block;background:#1e3a5f;color:#ffffff;padding:7px 16px;border-radius:5px;text-decoration:none;font-weight:bold;font-size:13px`. For AllJobs / Drushim / JobMaster / LinkedIn add a small grey note after it: "(log in to the board to send your CV)".
  - Before sending, check that every card has an Apply button and that no `apply_url` points to jobify360 or another mirror. If a job has no valid `apply_url`, fix it (see the table in `subagent_board_prompt.md`) or leave the job out.
  3. Footer: top 3 rejection reasons with counts, a pointer to `rejected_log.md`, and a reminder that keywords are edited in `keywords.md`.
- **No new matches:** send a short email with subject `Daily job matches – <date> (0 new)` and the footer line (postings checked, top rejection reasons).
- Mark every emailed job as `sent` in `seen_jobs.json` only after `send_message` succeeds.
