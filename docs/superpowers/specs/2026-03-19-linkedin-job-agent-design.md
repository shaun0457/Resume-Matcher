# LinkedIn Job Agent — Design Spec

**Date:** 2026-03-19
**Status:** Approved
**Companion project to:** Resume Matcher

---

## Overview

A semi-automatic job application pipeline that:
1. Scrapes LinkedIn job listings via Apify API on a schedule or on-demand
2. Deduplicates against previously-seen jobs using SQLite
3. For each new job, calls the Resume Matcher REST API to preview a tailored resume
4. Notifies the user via Telegram Bot with a summary and confirm/skip buttons
5. On confirmation, persists the tailored resume and delivers a PDF link — manual application left to the user

---

## Architecture

```
config.yaml
    │
    ▼
┌─────────────┐     Apify API      ┌──────────────┐
│  Scheduler  │ ─── fetch jobs ──► │   Scraper    │
│  (APSched)  │                    │  (Apify SDK) │
└─────────────┘                    └──────┬───────┘
       ▲                                  │ List[Job]
       │ /run                             ▼
┌──────┴──────┐                    ┌──────────────┐
│ Telegram    │                    │   Deduper    │ ◄── SQLite (seen_jobs)
│   Bot       │                    └──────┬───────┘
└─────────────┘                           │ new jobs only
                                          ▼
                                   ┌──────────────┐
                                   │   Improver   │ ◄── Resume Matcher API
                                   │              │     POST /api/v1/jobs/upload
                                   │              │     POST /api/v1/resumes/improve/preview
                                   └──────┬───────┘     (confirm only on user action)
                                          │ preview resume_id
                                          ▼
                                   ┌──────────────┐
                                   │   Notifier   │ ── Telegram Bot
                                   └──────────────┘    (inline buttons)
```

---

## Repository Structure

```
linkedin-job-agent/         ← new standalone repo
├── config.yaml
├── main.py                 # entry point: starts Bot + Scheduler
├── agent/
│   ├── scraper.py          # Apify API → List[Job]
│   ├── deduper.py          # SQLite dedup logic
│   ├── improver.py         # Resume Matcher API client
│   └── notifier.py         # Telegram Bot (messages + callbacks)
├── models.py               # Job, TailoredResult dataclasses
├── db.py                   # SQLite wrapper (stdlib sqlite3, no extra dep)
├── pyproject.toml          # managed with uv
├── .env.example
└── README.md
```

---

## Configuration (`config.yaml`)

```yaml
search:
  keywords:
    - "Senior Backend Engineer"
    - "Python Engineer"
  location: "Taiwan"
  experience_level:
    # Valid Apify actor values:
    # INTERNSHIP, ENTRY_LEVEL, ASSOCIATE, MID_SENIOR_LEVEL, DIRECTOR, EXECUTIVE
    - "MID_SENIOR_LEVEL"
    - "DIRECTOR"
  blacklist_companies: []     # exact company name strings
  max_jobs_per_run: 20

schedule:
  # APScheduler CronTrigger fields (separate values, not a cron string)
  hour: 8
  minute: 0
  # Runs every day at 08:00 local time

resume_matcher:
  base_url: "http://localhost:8000"
  # master resume is fetched at startup via:
  #   GET /api/v1/resumes/list?include_master=true
  # If no master resume is found, the agent logs an error and exits.
```

Secrets in `.env`:
```
APIFY_TOKEN=...
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
```

---

## Data Models

```python
@dataclass
class Job:
    job_id: str          # Apify-provided unique id
    title: str
    company: str
    location: str
    url: str
    description: str     # full JD text, used for Resume Matcher upload
    salary: str | None
    posted_at: str | None

@dataclass
class TailoredResult:
    job: Job
    preview_resume_id: str   # Resume Matcher preview resume id (not yet confirmed)
    keywords_added: list[str]
    # pdf_url is derived at notification time:
    #   f"{base_url}/api/v1/resumes/{preview_resume_id}/pdf"
    # and only valid after confirm is called
```

---

## SQLite Schema

```sql
-- stdlib sqlite3, no extra dependency
CREATE TABLE seen_jobs (
    job_id          TEXT PRIMARY KEY,
    title           TEXT NOT NULL,
    company         TEXT NOT NULL,
    url             TEXT NOT NULL UNIQUE,   -- secondary dedup key by URL
    status          TEXT NOT NULL DEFAULT 'notified',
    -- status values: 'notified' | 'confirmed' | 'skipped'
    preview_resume_id  TEXT,    -- set after improve/preview succeeds
    confirmed_resume_id TEXT,   -- set after improve/confirm succeeds (on user action)
    notified_at     TEXT NOT NULL,   -- ISO8601
    decided_at      TEXT             -- set when user confirms/skips
);
```

**Deduplication logic:** a job is considered seen if either its `job_id` OR its `url` already exists in `seen_jobs`. This handles Apify re-indexing the same LinkedIn listing with a different ID.

---

## Improve Endpoint Strategy

The pipeline uses a **preview → confirm** pattern to keep the user in control:

1. **On scrape** (background, before notification):
   - Call `POST /api/v1/resumes/improve/preview`
   - Stores `preview_resume_id` in `seen_jobs`
   - Resume exists in Resume Matcher but is not confirmed/persisted

2. **On Telegram ✅ confirm**:
   - Call `POST /api/v1/resumes/improve/confirm` with the `preview_resume_id`
   - Stores `confirmed_resume_id` in `seen_jobs`
   - Send PDF link: `GET /api/v1/resumes/{confirmed_resume_id}/pdf`

3. **On Telegram ❌ skip**:
   - Call `DELETE /api/v1/resumes/{preview_resume_id}` to clean up the preview
   - Status → `'skipped'`, `preview_resume_id` nulled

---

## Master Resume Lookup

At startup (and before each run), the agent calls:
```
GET {base_url}/api/v1/resumes/list?include_master=true
```
- If a master resume is found, its `id` is cached in memory
- If no master resume exists, the agent sends a Telegram error message and skips the run:
  ```
  ⚠️ No master resume found in Resume Matcher. Please upload one first.
  ```

---

## Resume Processing Wait

`POST /api/v1/jobs/upload` and related calls may return before processing completes.
After uploading a job description, the improver polls:
```
GET /api/v1/resumes/{id}  →  check .processing_status
```
- Poll every 3 seconds, up to 60 seconds (20 attempts)
- Terminal states: `completed`, `failed`
- If timeout or `failed`: skip this job, log error, continue to next

---

## Pipeline Steps (per run)

1. **Fetch master resume ID** — startup cache or fresh lookup
2. **Scrape** — Apify LinkedIn Jobs Scraper actor, returns `List[Job]`
3. **Deduplicate** — filter jobs where `job_id` OR `url` already in `seen_jobs`
4. **For each new job:**
   a. `POST /api/v1/jobs/upload` → `rm_job_id`
   b. `POST /api/v1/resumes/improve/preview` with `master_resume_id` + `rm_job_id`
   c. Poll until `processing_status == 'completed'` (or skip on timeout/fail)
   d. Extract `improvements[]` diff for notification summary
   e. Insert row in `seen_jobs`: `status='notified'`, `preview_resume_id` set
5. **Notify** — Telegram message per job (see format below)

---

## Telegram Bot

### Callback Data

`callback_data` for inline buttons is a compact string:
```
confirm:{job_id}
skip:{job_id}
```
The handler looks up `preview_resume_id` from `seen_jobs` by `job_id`.

### Message Format (per job)

```
🏢 *{company}* — {title}
📍 {location}
🔗 {url}

📄 履歷調整重點：
• 新增關鍵字：{keywords_added joined by ", "}
• 調整 {n} 項工作描述

[✅ 確認]  [❌ 跳過]
```

### On ✅ confirm

1. Call `POST /api/v1/resumes/improve/confirm` → `confirmed_resume_id`
2. Update `seen_jobs`: `status='confirmed'`, `confirmed_resume_id` set, `decided_at` set
3. Edit message to remove buttons, reply:
   ```
   ✅ 履歷已確認
   📥 下載 PDF → {base_url}/api/v1/resumes/{confirmed_resume_id}/pdf
   ```
4. **On failure:** reply with `⚠️ 確認失敗，請稍後再試 (/retry {job_id})`; status remains `'notified'`

### On ❌ skip

1. Call `DELETE /api/v1/resumes/{preview_resume_id}` (best-effort, log if fails)
2. Update `seen_jobs`: `status='skipped'`, `decided_at` set
3. Edit message: replace content with `~~{title} @ {company}~~ — 已跳過`

### Commands

| Command | Action |
|---------|--------|
| `/run` | Immediately execute one pipeline run |
| `/status` | Today's stats: found / confirmed / skipped |
| `/list` | Last 10 confirmed jobs (title, company, pdf link) |
| `/retry {job_id}` | Retry confirm for a job stuck in `'notified'` state |

*Note: `/status` replaces a `/today` command — Telegram commands cannot contain spaces.*

---

## Error Handling

| Error | Behaviour |
|-------|-----------|
| Apify auth / rate limit | Log, notify user via Telegram: `⚠️ Scraper failed: {reason}`, skip run |
| No master resume found | Notify user, skip run |
| Resume Matcher unreachable | Log, skip individual job, continue to next, summarise failures at end |
| `improve/preview` returns error | Log, skip job, do NOT insert into `seen_jobs` (will retry next run) |
| Processing poll timeout | Log, skip job, do NOT insert into `seen_jobs` |
| Telegram send failure | Retry 3× with exponential backoff (2s, 4s, 8s), then log |
| Confirm callback failure | Reply with error message, status stays `'notified'`, user can `/retry` |
| Skip cleanup failure | Log but continue — DELETE is best-effort; orphaned preview resumes are acceptable |

---

## Dependencies

```toml
[project]
requires-python = ">=3.11"
dependencies = [
    "apify-client>=1.7",
    "python-telegram-bot>=21.0",
    "apscheduler>=3.10",
    "httpx>=0.27",
    "pyyaml>=6.0",
    "python-dotenv>=1.0",
]
# sqlite3 is Python stdlib — no extra dependency needed
```

---

## Out of Scope (v1)

- Auto-submitting applications
- Multiple resume profiles per run
- Dynamic search config via Telegram
- Email notifications
- Web dashboard

---

## Definition of Done

- [ ] `uv run python main.py` starts Bot + Scheduler without error
- [ ] `/run` command finds, deduplicates, improves (preview), and notifies for ≥1 job
- [ ] Confirm button calls `improve/confirm` and delivers a valid PDF link
- [ ] Skip button deletes the preview resume and marks the job skipped
- [ ] Re-running pipeline does not re-notify already-seen jobs (by id or URL)
- [ ] README documents full setup: Apify token, Telegram bot creation, Resume Matcher URL
