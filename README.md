![Node.js](https://img.shields.io/badge/Node.js-scraper-339933?logo=nodedotjs&logoColor=white)
![Supabase](https://img.shields.io/badge/sync-Supabase-3ECF8E?logo=supabase&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/runs%20on-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

# aisis-scraper

**Keeps a copy of Ateneo's AISIS class schedules and degree curricula in Supabase, refreshed automatically.**

AISIS is the Ateneo de Manila student information system. This scraper logs in with an AISIS account, reads the *Schedule of Classes* and *Official Curriculum* pages that any logged-in student can see, and syncs them into a Supabase database — so other apps can use the course catalogue without each one scraping AISIS.

- **What it collects:** course offerings per term (course code, title, section, schedule, room, instructor name, slots) and the official curriculum of each degree program. It does not read student records or grades.
- **How often:** GitHub Actions run it on a schedule — class schedules every 6 hours (current and next term), all terms and curricula weekly, a full academic year on demand.
- **How it stays safe:** before replacing data it compares each department's course count with the last good run, and refuses to sync if a department suddenly drops (a sign AISIS returned a bad page). It can also write to Google Sheets.

Curriculum scraping is experimental; class schedules are the stable path.

## Quick start

```bash
npm install
cp .env.example .env    # then fill in the four keys below
npm start               # scrape class schedules
```

| Command | What it does |
| --- | --- |
| `npm start` / `npm run dev` | scrape class schedules and sync them |
| `npm run curriculum` | scrape degree curricula (experimental) |
| `npm run verify` | check that every AISIS schedule made it into Supabase |
| `npm run validate:subjects` | per-department course and subject-prefix counts from the last scrape |
| `npm test` | parser tests |
| `npm run test:all` | parser + PE subject tests |

The Supabase side (Edge Functions in `supabase/functions/`, schema in `supabase/migrations/`) has to be deployed first — see [docs/SETUP.md](docs/SETUP.md).

## Configuration

Required:

- `AISIS_USERNAME`, `AISIS_PASSWORD` — the AISIS account to log in with.
- `SUPABASE_URL` — the Supabase project that receives the data.
- `DATA_INGEST_TOKEN` — token for the Supabase ingest Edge Function.

Optional (most used):

- `AISIS_SCRAPE_MODE` — `current`, `current_next` (default), `future`, `all` or `year`.
- `AISIS_TERM` — force a term such as `2025-1` instead of auto-detecting.
- `AISIS_DEPARTMENTS` — only scrape these departments (comma-separated).
- `GOOGLE_SERVICE_ACCOUNT`, `SPREADSHEET_ID` — also write to Google Sheets.

There are about thirty more tuning keys (concurrency, delays, batch sizes, regression thresholds, debug flags). Every one is listed with its default in [docs/internals.md](docs/internals.md#environment-variables-summary). In CI the same keys are set as GitHub Actions secrets.

## How it works

```
GitHub Actions (cron) ──▶ src/index.js ──▶ AISIS (login, fetch HTML) ──▶ cheerio parser
                                                                            │
                         baseline check per department ◀────────────────────┘
                                     │ ok
                                     ▼
                    Supabase Edge Function (github-data-ingest) ──▶ Postgres
                                     └──▶ Google Sheets (optional)
```

It uses plain HTTP requests (`node-fetch`) and `cheerio`, not a headless browser, so it is fast and light. Large syncs are batched on both the client and the Edge Function side to avoid timeouts.

## Links

- [docs/internals.md](docs/internals.md) — the full previous README: performance tuning, batching architecture, baselines, data-loss protection, troubleshooting
- [docs/](docs/) — setup, deployment, data guide, Google Sheets guide, ingestion, term codes
- [MULTI_TERM_SCRAPING.md](MULTI_TERM_SCRAPING.md) — the scrape modes in detail
