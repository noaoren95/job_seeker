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
  ├─ 5 board subagents in parallel (subagent_board_prompt.md)
  │    linkedin · alljobs · drushim · jobmaster · career_pages
  │    search → open postings → requirements, posted date, apply link
  ├─ merge, remove duplicates, skip seen jobs, keep only jobs from the last N days
  ├─ CV-fit gate: every must-have needs a matching CV line (+ seniority, degree, location, role fit)
  ├─ score 0–10, recommend ≥ 7
  └─ HTML email via the Gmail connector (Apply button per job) → update seen_jobs.json
```
The CV is read only by the lead agent. It is never sent to subagents or put in search queries.

## Setup
1. Copy `cv.example.md` to `cv.md` and replace it with your own CV. `cv.md` is git-ignored.
2. Put your target job description in `job_description.txt`.
3. Ask Claude to run the extractor (`extractor_prompt.md`) on both files. This creates `profiles/` and `keywords.md`.
4. In Claude (desktop app), connect the Gmail connector and create a scheduled task that runs the lead agent (`agent_prompt.md`) on your chosen schedule, e.g. 08:00 Sunday–Thursday.

Scheduled tasks run only while the Claude desktop app is open. A run that was missed happens the next time the app starts.

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
| `runs/<date>/` | Each run's board results, results.md, email.html | **No** (ignored) |

## Common changes
- **Add/remove keywords:** edit `keywords.md` (put `~` before a term to turn it off, or move it to `## Removed`), or tell Claude "add X to core" / "disable Y".
- **How recent jobs must be:** in `keywords.md` → `## Filters`, set `Max job age (days)` to any number (3, 7, 10…). Set `Unknown posting date` to `include` to also keep jobs without a confirmed date.
- **New target job:** replace or add the job description and ask Claude to "re-extract keywords from the new JD". Only the job-description side and the merge are re-run, and your manual edits are kept.
- **Updated CV:** replace `cv.md` and ask Claude to "re-extract the CV profile".
- **Stricter/looser matching:** change the score threshold (7) or the gate rules in `agent_prompt.md`.
- **Change time/days:** edit the scheduled task in the Claude app's Scheduled section.
