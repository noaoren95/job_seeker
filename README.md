# Job Seeker Agent

An AI agent (built on Claude) that searches Israeli job boards every workday using **ATS keywords**, keeps only the jobs your **CV actually backs up**, and emails them to you with an Apply link for each one.

## How it works
```
         one-time, on request
job description ─► extractor ─► jd_profile ─┐
CV ─────────────► extractor ─► cv_profile ─┴─► merge ─► keywords.md (you edit this)
                                                             │ read every run
                                                             ▼
LEAD AGENT (agent_prompt.md)
  ├─ builds queries for each board from keywords.md
  ├─ 6 board subagents in parallel (subagent_board_prompt.md)
  │    linkedin · indeed · alljobs · drushim · jobmaster · career_pages
  │    search → open postings → requirements, posted date, apply link
  ├─ merge, remove duplicates, skip seen jobs, keep only jobs from the last N days
  ├─ CV-fit gate: every must-have needs a matching CV line (+ seniority, degree, location, role fit)
  ├─ score 0–10, recommend ≥ 7
  ├─ HTML email via the Gmail connector (Apply button per job) → update seen jobs
  └─ progress log + checkpoints (runs/<date>/)
```
The CV is read only by the lead agent. It is never sent to subagents or put in search queries.

## Prerequisites
| What | Why |
|---|---|
| **Claude desktop app** (Windows or macOS) with the **Code** tab (Claude Code), on a paid Claude plan | Runs the agents, the parallel subagents, and the daily **scheduled task** |
| **Gmail connector** (Google account) connected in Claude | Sends the daily email (`send_message`) |
| **Web search + web fetch** allowed in Claude Code | The board subagents search the job boards and open postings |
| **Git** (optional) | To clone this repo |
| A computer that's **on, with the Claude app open**, at the scheduled time | Scheduled tasks run locally; a missed run happens the next time the app starts |

No API keys, Python or other installs are needed. Claude is the LLM, and everything else is markdown instructions.

## First-time setup
1. **Get the project**
   ```bash
   git clone https://github.com/noaoren95/job_seeker.git
   ```
   Or download it as a ZIP from GitHub and unzip it.

2. **Open it in Claude.** In the Claude desktop app, go to the **Code** tab, start a new session, and pick the `job_seeker` folder as the working folder.

3. **Connect Gmail.** In the Claude app, open **Settings → Connectors**, add **Gmail**, and sign in with the Google account that should receive the emails. Make sure the connector is turned on for your Code sessions.

4. **Add your CV.** Copy `cv.example.md` to a new file named `cv.md` in the same folder, then replace its content with your own CV (Markdown or plain text; keep the same sections where you can). `cv.md` is git-ignored, so it is never committed.

5. **Add your target job.** Paste a job description that describes the role you want into `job_description.txt`, replacing the example.

6. **Extract your keywords.** In the Claude session, send:
   > Follow extractor_prompt.md: extract a profile from job_description.txt and from cv.md, then merge them into keywords.md.

   This creates `profiles/jd_profile.md`, `profiles/cv_profile.md` and a fresh `keywords.md`.

7. **Review `keywords.md`.** Remove terms you don't want (or disable them with `~`), add your own under `## My additions`, and set the **Filters**: location, seniority, `Max job age (days)`, and `Unknown posting date`.

8. **Do a test run.** In the Claude session, send:
   > Run the lead agent in agent_prompt.md once now and email the results to me.

   Approve the web and Gmail permission prompts when they appear. You should get an email within about 10–15 minutes. Check the results, then tune `keywords.md` or the rules in `agent_prompt.md` if needed.

9. **Schedule it.** In the Claude session, send (replace the two `<…>` values):
   > Create a scheduled task named daily-job-seeker that runs at 08:00 Sunday–Thursday. Each run: act as the lead agent in `<full path to job_seeker>\agent_prompt.md`, follow it exactly, and send the email with the Gmail connector to `<your email>`.

   You can also create or edit it later from **Scheduled** in the app's sidebar.

10. **Avoid permission pop-ups (optional).** A scheduled run happens while you're away. If it stops to ask for permission, it waits until you approve. Either save the task in **Auto** permission mode, or allow `WebSearch`, `WebFetch` and the Gmail connector in this project's Claude Code permissions.

`seen_jobs.json`, `rejected_log.md` and `runs/` are created automatically on the first run, and git ignores them.

## Cloud schedule (Claude routine)
A local scheduled task only runs while your computer and the Claude app are open. To run even when they're closed, use a **Claude routine** (a cloud session on a schedule) instead, for example `job-seeker-daily` with cron `0 7 * * *` (UTC): 09:00 Israel winter time, 10:00 in summer.

How cloud mode works:
- Each run starts from a fresh clone of this repo, so it has the prompts and `keywords.md`, but **not** your private files.
- Your private files stay in **Google Drive**, in the project folder: `cv.md`, `seen_jobs.json`, `state/seen/`, `runs/`, `logs/`. The run reads and writes them through the **Google Drive connector** and sends the email through the **Gmail connector**. Both must be connected at claude.ai → Settings → Connectors.
- The Drive connector can only create files, not overwrite them, so each cloud run writes its seen jobs to a new `state/seen/<date>.json`. The agent (local or cloud) combines these with `seen_jobs.json`.
- The routine prompt (not this repo) holds your email address and the Drive folder IDs.
- Prompt changes only reach the cloud once they're **pushed to GitHub**.
- After the cloud routine is working, disable the local scheduled task. (If both run, the "already sent today" check stops a second email.)
- Cron is in UTC: update it when daylight saving time changes, if you want an exact local time.

### Cloud network access (required)
Cloud environments start with **limited** network access, so the job boards are blocked (`CONNECT tunnel failed, response 403` / `getaddrinfo ENOTFOUND`) and every board comes back empty. To fix it, go to claude.ai/code, open the cloud environment the routine uses, and under **Network access** choose:
- **Full**: simplest, since the career-pages board visits company sites that change every day, or
- **Custom**, allowing at least: `linkedin.com`, `il.linkedin.com`, `il.indeed.com`, `alljobs.co.il`, `drushim.co.il`, `jobmaster.co.il`, `jobify360.co.il` (the mirror the boards' full text comes from), `boards-api.greenhouse.io`, `job-boards.greenhouse.io`, `job-boards.eu.greenhouse.io`, `jobs.lever.co`, `jobs.workable.com`, `apply.workable.com`, `smartrecruiters.com`, `builtin.com`, `jobs.tlv.partners`, `careers.grovevc.com`, `jobs.detroit.vc`.

Each cloud run checks the network first (`agent_prompt.md` section 0d). If every board is blocked, it skips the search and sends an email marked "⚠ network blocked".

Manage routines at claude.ai/code/routines, or ask Claude ("run the job-seeker-daily routine now", "show the last run's log").

## Logging
Every run leaves a trail you can check:
| Where | What |
|---|---|
| `runs/<date>/progress/` | Checkpoints: `01_started` → `02_boards_done` → `03_scored` → `04_email_sent` or `04_failed`. The last file shows how far a run got. `r_` = retry |
| `runs/<date>/run_log.jsonl` | One line per event (start, each board, merge, gate, email, finish, errors), with timestamps |
| `logs/history/<date>.md` | One summary line per run: start/end time, checked, sent, email ✓/✗, attempt |

Each local scheduled run's full session is also listed under **Scheduled** in the Claude app's sidebar; a cloud routine's runs are on its page at claude.ai/code/routines.

## Files
| File | What it is | In git? |
|---|---|---|
| `keywords.md` | Live keyword list + filters (job age, location, seniority). **Edit freely** | Yes |
| `cv.example.md` | Fictional example CV showing the expected format | Yes |
| `cv.md` | **Your** CV, the basis of the match gate | **No** (ignored) |
| `job_description.txt` | The target job description | Yes |
| `extractor_prompt.md` | The single keyword extractor (job description and CV) | Yes |
| `profiles/jd_profile.md` | Keywords extracted from the job description | Yes |
| `profiles/cv_profile.md` | Keywords extracted from your CV | **No** (ignored) |
| `agent_prompt.md` | Lead agent: queries, freshness, CV gate, scoring, email | Yes |
| `subagent_board_prompt.md` | Board subagent: per-site search rules, apply links | Yes |
| `templates/email_template.html` | Reference layout for the daily email | Yes |
| `seen_jobs.json` | Jobs already handled (prevents repeats) | **No** (ignored) |
| `rejected_log.md` | Last run's rejected jobs + reasons | **No** (ignored) |
| `runs/<date>/` | Each run's board results, results.md, email.html, run_log.jsonl, progress/ | **No** (ignored) |
| `state/seen/<date>.json` | Jobs handled by each cloud run (added to the seen list) | **No** (ignored) |
| `logs/history/` | One summary line per run | **No** (ignored) |

## Common changes
- **Add/remove keywords:** edit `keywords.md` (put `~` before a term to turn it off, or move it to `## Removed`), or tell Claude "add X to core" / "disable Y".
- **How recent jobs must be:** in `keywords.md` → `## Filters`, set `Max job age (days)` to any number (3, 7, 10…). Set `Unknown posting date` to `include` to also keep jobs without a confirmed date.
- **New target job:** replace or add the job description and ask Claude to "re-extract keywords from the new JD". Only the job-description side and the merge are re-run, and your manual edits are kept.
- **Updated CV:** replace `cv.md` and ask Claude to "re-extract the CV profile".
- **Stricter/looser matching:** change the score threshold (7) or the gate rules in `agent_prompt.md`.
- **Change time/days:** edit the scheduled task in the Claude app's Scheduled section.
