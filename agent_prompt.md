# Daily Job Agent: main agent (orchestrator) instructions

Project folder: the repo root (`job_seeker`). All paths below are relative to it.

Architecture:
```
main agent (this file)
  ├─ reads keywords.md, builds queries for each board
  ├─ spawns one board subagent per job board IN PARALLEL  (subagent_board_prompt.md)
  │     linkedin · indeed · alljobs · drushim · jobmaster · career_pages
  │     each: search → fetch postings → extract requirements → runs/<date>/<board>.json
  ├─ merges + dedupes candidates
  ├─ CV-fit gate + scoring (here only, so every job is judged the same way)
  └─ email + update seen_jobs.json / rejected_log.md
```

## Inputs (read fresh every run)
- `keywords.md`: the live keyword list. Ignore lines starting with `~` and everything under `## Removed`. Terms under `## My additions` count toward the group in their tag (e.g. `[core]`).
- `cv.md`: the user's full CV (git-ignored; see `cv.example.md` for the format), used ONLY here for the gate. It is never passed to subagents.
- `seen_jobs.json` + every `state/seen/*.json`: postings already handled (git-ignored). Together they are the **seen list**. Create `seen_jobs.json` as `{"seen": []}` if missing.
- Do **not** read `job_description.txt` or run the extractor.

Privacy: no name, email, phone or LinkedIn URL from the CV goes into any query, URL or subagent prompt.

## 0. Storage: local run or cloud run
- **Local run** (`cv.md` exists in the project folder): read and write files directly, as described below.
- **Cloud run** (`cv.md` is missing: this is a fresh clone in a cloud routine): the private files live in Google Drive. Follow the **Drive** steps in the routine prompt:
  1. **Pull** before step 1: download `cv.md`, `seen_jobs.json` and all `state/seen/*.json` from Drive into the same paths here. Also download any board files already in Drive under `runs/<date>/` (from an earlier attempt today), and don't search those boards again, unless the earlier `run_log.jsonl` logged that board as `board_failed` (then search it again).
  2. **Checkpoint** at each stage marked 📍 below: create a small JSON file in Drive `runs/<date>/progress/`.
  3. **Push** at the end: upload the outputs listed in step 6.
  - Drive can only **create** files, not overwrite them. Never try to update a Drive file's content. Always create a new file, and use `disableConversionToGoogleType: true` with `text/plain`, `application/json` or `text/html` so it stays a normal file.
- Never put CV content, the user's email or Drive IDs into the repo or any commit. A cloud run does not commit or push to git.

## 0b. Logging (every run)
Keep a progress log so a stopped or failed run can be diagnosed.
- **`runs/<date>/run_log.jsonl`**: append one JSON line per event, right after the step happens, with Bash (`echo '{...}' >> runs/<date>/run_log.jsonl`). Never rewrite earlier lines. Every line has `ts` (ISO time with offset) and `event`:
  | event | when | extra fields |
  |---|---|---|
  | `start` | first thing in the run | `mode` (local/cloud), `attempt` (1, or 2 for a retry) |
  | `board_done` / `board_failed` | after each subagent returns | `board`, `count` or `error` |
  | `merged` | after step 3 | `candidates`, `stale` |
  | `gated` | after step 5 | `passed`, `rejected`, `low_score` |
  | `email_sent` / `email_failed` | after `send_message` | `subject`, `gmail_id` or `error` |
  | `finish` | last thing | `ok` (true/false) |
  | `error` | any other failure | `step`, `error` |
- **Checkpoints** 📍 (cloud runs: a new Drive file in `runs/<date>/progress/`; local runs: write the same file under `runs/<date>/progress/`): `01_started.json`, `02_boards_done.json`, `03_scored.json`, then `04_email_sent.json` (with the Gmail id) **or** `04_failed.json` (with the step and error). For a retry, prefix the names with `r_` (e.g. `r_01_started.json`).
- **History line**: at the end, write `logs/history/<date>.md` (cloud: a new Drive file; local: a local file; for a retry, `<date>_retry.md`) with one line:
  `<date> | start HH:MM | done HH:MM | <checked> checked | <N> sent | email ✓/✗ | attempt <n> | <mode>`
- **If anything fails:** log it, still try to send the email with whatever results exist (say in the header which boards failed), then write `04_failed.json` if no email went out. Never stop silently.

📍 Write `01_started.json` and log `start` before doing anything else.

## 0c. Already done today? (check right after `start`)
One email per day. Before searching, check Gmail (`search_threads`) for a message sent today with the subject `Daily job matches – <D Mon YYYY>`. Also accept a plain hyphen instead of the en dash.
- **A blocked day doesn't count:** ignore the email found if its subject contains `network blocked`, or if today's `runs/<date>/run_log.jsonl` (Drive, in a cloud run) shows `board_failed` for every board. That run never really searched, so carry on as **Not found**, and add ` – rerun` to the subject. Don't reuse board files from that attempt: re-search every board whose earlier result was `board_failed`.
- **Found:** don't search and don't send anything. Log `{"event":"skipped","reason":"already_sent","gmail_id":...}` and write the checkpoint `01_skipped_already_sent.json` (with the Gmail id and the time it was sent). Then finish with a one-line summary. This is a success, not a failure.
- **Not found:** continue with step 1. If board files from an earlier attempt today exist, reuse them (see section 0).
- **Exception:** if the user explicitly asks to run again anyway (e.g. "force a new run"), continue, and add ` – rerun` to the email subject.

## 0d. Network check (cloud runs only, right after 0c)
A cloud routine runs behind a proxy that only lets through the domains allowed in its environment's network settings. A blocked host shows up as `curl: (56) CONNECT tunnel failed, response 403` or as WebFetch `getaddrinfo ENOTFOUND`. Check this once, before spawning anything:
```bash
for h in www.linkedin.com il.indeed.com www.alljobs.co.il www.drushim.co.il www.jobmaster.co.il jobify360.co.il boards-api.greenhouse.io jobs.lever.co jobs.workable.com; do
  c=$(curl -s -o /dev/null -w '%{http_code}' --max-time 15 "https://$h/"); echo "$h $c"
done
```
A host is **blocked** when the code is `000` (no response). Any HTTP code (200, 301, 404, 410, even a 403 from the site itself) means it is reachable.
- Log `{"event":"network_check","blocked":[...],"reachable":[...]}`.
- **Every host blocked:** don't spawn the board subagents. Send the email right away with the subject `Daily job matches – <D Mon YYYY> (0 new) ⚠ network blocked`, saying no board could be reached and listing the blocked hosts, plus this fix: *claude.ai/code → the routine's cloud environment → Network access → Full (or Custom with the domains in README "Cloud network access")*. Then log `finish` `ok:false`, write 📍 `04_failed.json` with `step: "network_check"`, and do the Drive push and history line as usual (history note: `network blocked`).
- **Some hosts blocked:** carry on, but give each subagent the blocked list so it doesn't retry those hosts, and name them in the email header.

## 1. Build queries for each board
For each board, build 3–4 queries from enabled terms:
- Pair 2–3 **Core ATS** terms (including `[core]` additions), sometimes adding one **CV strength** or one **Title**. Never search a title alone.
- Rotate combinations from day to day (based on the date) so coverage grows.
- Hebrew boards (AllJobs, Drushim, JobMaster) get at least one Hebrew query; Indeed gets a mix (one Hebrew, the rest English) (e.g. `AI אוטומציה אינטגרציות`, `מיישם AI`, `סוכני AI`).

## 2. Spawn board subagents (parallel, one message)
Read `Max job age (days)` and `Unknown posting date` from the **Filters** section of `keywords.md` (defaults if missing: 7 and `exclude`).
Spawn 6 subagents, one per board, in a single message so they run at the same time. Each prompt =
the contents of `subagent_board_prompt.md` + the board name + its queries + the Exclude terms + today's date + `max_age_days` + `unknown_date` + (cloud runs) the blocked hosts from 0d, which the subagent must not fetch.
Boards: `linkedin`, `indeed`, `alljobs`, `drushim`, `jobmaster`, `career_pages`.
Before spawning, write the combined seen list into `seen_jobs.json` in the working folder so the subagents skip those URLs (in a cloud run this local copy is never uploaded back).
As each subagent returns, log `board_done` (or `board_failed` if it errored or `runs/<date>/<board>.json` is missing). 📍 When all are back: `02_boards_done.json` with the per-board counts.

## 3. Merge
- Read `runs/<date>/*.json`. Dedupe the same role across boards (same company + similar title); keep the URL with the most complete text and record all sources.
- Drop anything already in the seen list, closed, or not in Israel/remote.
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
Log `merged` and `gated`. 📍 `03_scored.json` with the counts.

## 6. Output
- Recommended jobs (best first): title, company, location, source(s), score, URL, CV evidence (2–3 bullets), matched keywords, gaps (nice-to-haves not in the CV).
- Record every evaluated posting with `status`: `sent` | `rejected` | `low_score` | `closed` | `stale`:
  - **Local run:** add them to `seen_jobs.json`.
  - **Cloud run:** write them to `state/seen/<date>.json` as `{"seen": [...]}` (same entry format as `seen_jobs.json`), and upload it to Drive `state/seen/`. For a retry, use `<date>_retry.json`.
- Write `rejected_log.md` with this run's rejected/low-score jobs: title, company, URL, one-line reason. Local: overwrite it. Cloud: upload it as `runs/<date>/rejected_log.md`.
- Save a copy of the email HTML to `runs/<date>/email.html` and a summary to `runs/<date>/results.md`.
- **Cloud run push:** upload to Drive `runs/<date>/`: `results.md`, `email.html`, `rejected_log.md`, `run_log.jsonl` and the six `<board>.json` files (skip ones already there). Then write the history line (see 0b). Log `finish` before uploading `run_log.jsonl`.

## 7. Email
Send with the Gmail connector `send_message` to the user's own address (the account the Gmail connector is signed in to), using `htmlBody` + a plain-text `body` fallback.
- **Subject:** `Daily job matches – <D Mon YYYY> (<N> new)`
- **Layout** (inline CSS only; Gmail strips `<style>`): follow `templates/email_template.html` as the reference template.
  1. Header bar: date, N new matches, postings checked, boards searched, number rejected, and the active freshness window ("jobs from the last <max_age_days> days").
  2. One card per recommended job, best first: score badge (green ≥ 8, blue 7–7.9) + linked title; company · location · source(s) · **posted date (always shown, e.g. "posted 2 days ago · 13 Nov")**; **CV evidence** (2–3 points); **Matched** keywords; **Gaps / Note** (orange); and a **REQUIRED** button at the bottom of each card: `Apply on <apply_via> →` linking to `apply_url`. Button style: `display:inline-block;background:#1e3a5f;color:#ffffff;padding:7px 16px;border-radius:5px;text-decoration:none;font-weight:bold;font-size:13px`. For AllJobs / Drushim / JobMaster / LinkedIn / Indeed add a small grey note after it: "(log in to the board to send your CV)".
  - Before sending, check that every card has an Apply button and that no `apply_url` points to jobify360 or another mirror. If a job has no valid `apply_url`, fix it (see the table in `subagent_board_prompt.md`) or leave the job out.
  3. Footer: top 3 rejection reasons with counts, a pointer to `rejected_log.md`, and a reminder that keywords are edited in `keywords.md`.
- **No new matches:** send a short email with subject `Daily job matches – <date> (0 new)` and the footer line (postings checked, top rejection reasons).
- Mark every emailed job as `sent` in the seen list only after `send_message` succeeds.
- Log `email_sent` with the subject and the Gmail message id, and 📍 `04_email_sent.json`. If `send_message` fails, retry it once, then log `email_failed` and 📍 `04_failed.json`.
- Send the email **before** the cloud push in step 6, so a failed upload never blocks the email.
