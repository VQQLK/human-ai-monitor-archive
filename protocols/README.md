# Protocols archive

Weekly protocol snapshots, synced from production API.

## Source of truth

The production Cloudflare D1 database (table `protocols`, column `content`) is the source of truth.
The API reads directly from D1:

- `GET /protocols` — list of all protocols (returns both final and interim)
- `GET /protocols/{week_start}/content` — markdown for a specific week
- `GET /protocols/current` — live draft of current (still-open) week (NOT saved to DB)

Files in this directory are read-only snapshots, refreshed automatically by
`.github/workflows/sync-protocols.yml` (runs Monday and Friday at 08:00 UTC).

## File naming

`{week_end}.md`, where `week_end` is the **Sunday** of the covered week.
The covered period runs from the **Monday** 6 days before `week_end` through
the **Sunday** `week_end` itself.

Example: `2026-09-28.md` covers Monday 2026-09-22 -> Sunday 2026-09-28.

| File | Monday (API: week_start) | Sunday (file: week_end) | Coverage |
|------|--------------------------|-------------------------|----------|
| 2026-09-28.md | 2026-09-22 | 2026-09-28 | Mon 22 Sep -> Sun 28 Sep |
| 2026-10-05.md | 2026-09-29 | 2026-10-05 | Mon 29 Sep -> Sun 05 Oct |
| 2026-10-12.md | 2026-10-06 | 2026-10-12 | Mon 06 Oct -> Sun 12 Oct |

### Why Sunday-based naming for files?

- "The week ending September 28" is more intuitive for human readers than
  "the week starting September 22"
- Matches how people naturally think about "last week" (as a completed period)

### Note on API identifier

The production API and database use the **Monday** (week_start) as the
canonical identifier. For example:

- API endpoint: GET /protocols/2026-09-22/content
- Database: protocols.week_start = "2026-09-22"

This archive uses **Sunday** (week_end) only for file names to be more
human-readable. The mapping is straightforward:

    file_date (Sunday) = api_date (Monday) + 6 days

So 2026-09-28.md corresponds to API endpoint /protocols/2026-09-22/content.

## Architecture v2 (since 2026-09-23)

### Data collection

Twice daily, 5 cron triggers within Cloudflare Free plan limits:

| UTC time | New York | London | Moscow | Beijing | Action |
|----------|----------|--------|--------|---------|--------|
| 13:00 | 09:00 | 14:00 | 16:00 | 21:00 | Batch 1/4 |
| 13:15 | 09:15 | 14:15 | 16:15 | 21:15 | Batch 2/4 |
| 13:30 | 09:30 | 14:30 | 16:30 | 21:30 | Batch 3/4 |
| 13:45 | 09:45 | 14:45 | 16:45 | 21:45 | Batch 4/4 |
| 23:00 | 19:00 | 00:00+1 | 02:00+1 | 07:00+1 | Evening batch |

Each batch: 8 sources x 3 articles = ~24 items, ~48 subrequests (under 50 limit).

### Protocol generation

**Two protocols per week:**

**Monday at 13:45 UTC**: FINAL protocol for previous week
- is_interim = 0
- Covers completed Monday-Sunday period (7 days of data)
- Published to GitHub as {week_end}.md

**Friday at 13:45 UTC**: INTERIM protocol for current week
- is_interim = 1
- Covers in-progress Monday-Friday period (5 days of accumulated data)
- Published to GitHub as {week_end}.interim.md
- Purpose: snapshot for operational analysis mid-week

**Draft preview**: /protocols/current endpoint (not saved to DB)
- **Translation**: Monday and Friday at 13:50 UTC via GitHub Actions (translate-protocols.yml)
  - Runs 5 minutes after protocol generation (13:45 UTC)
  - Translates English protocol to Russian and Chinese
  - Updates content_ru and content_zh columns in database
- **Publication**: Monday 14:00 UTC (after final) + Saturday 08:00 UTC (after interim) via GitHub Actions

### File naming for interim protocols

| Type | Filename | Example |
|------|----------|---------|
| Final | {week_end}.md | 2026-09-28.md |
| Interim | {week_end}.interim.md | 2026-10-04.interim.md |

Example:
- Monday 2026-09-29 13:45 UTC generates final protocol for week 2026-09-22 -> 2026-09-28
- File: 2026-09-28.md (covers 7 days of data: Mon-Sun)

- Friday 2026-10-02 13:45 UTC generates interim protocol for week 2026-09-28 -> 2026-10-04
- File: 2026-10-04.interim.md (covers 5 days of data: Mon-Fri)

### Why this architecture?

RSS feeds store only ~20-50 recent articles (approx 2-3 days of history).
With weekly collection (old schedule), articles published Wednesday-Monday
fell out of RSS before Tuesday's collection, causing 50-70% data loss.

Twice-daily collection ensures articles are collected within 12 hours
of publication, eliminating RSS "fallout" problem.

## Why this archive exists

- **Historical backup** — survives accidental D1 data loss
- **Grep-able history** — search across years without hitting the API
- **Public record** — anyone can browse protocol evolution on GitHub

## Manual recovery

If D1 is ever lost, the archive alone is not sufficient to rebuild — the
items table (with per-article axes, relevance, shift) is not mirrored
here. But the markdown protocols give a full historical record of what
the Gap Index looked like at each week, which is the primary output.
