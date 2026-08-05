# cassidy-calendar-feed-test

**This repo is a parallel experiment. Nothing in production reads it.**

The live TV calendars live in [`cassidy-tv-calendars`](https://github.com/cassidysmartsheet-ux/cassidy-tv-calendars)
and are completely untouched by anything here. This repo exists only to prove
out a faster refresh before we consider changing anything the crews rely on.

## The problem being tested

The live feed asks GitHub to refresh every 5 minutes. It does not.

Measured across all 1,109 runs since 2026-05-06:

| | asked for | actually delivered |
|---|---|---|
| refreshes per day | 288 | ~12 |
| gap between refreshes | 5 min | 94 min median (51 min best, 216 min worst) |

GitHub's scheduled trigger is explicitly best-effort and can be delayed or
dropped. For this repo it has never come close to 5 minutes, on any day, since
the workflow was created.

A second ceiling sits behind it: the live feed stamps `generatedAt` into the
file every run, so every run commits, so every run rebuilds the site. GitHub
Pages soft-caps site builds at 10/hour. Even a perfectly punctual 5-minute
schedule (12/hour) would have been throttled.

## What this tests instead

One job that **stays resident**: it refreshes every 5 minutes for a ~5h40m
shift (GitHub caps a job at 6h), then dispatches its own replacement. Cron is
demoted to a safety net that restarts the chain if a shift dies.

It commits **only when the schedule data actually changed**, ignoring the
timestamp, which keeps site rebuilds far below the ceiling.

## Smartsheet safety

This repo only ever issues `GET` against the Smartsheet Reports API. There is
no code path here that can create, update, or delete anything in Smartsheet.
The token used is the read-only `CalendarGit` token, and it is stored as an
encrypted Actions secret rather than in plaintext.

## Reading the results

- `data-v2.json` — the test output. Compare its commit times against
  `data.json` in the live repo.
- Actions run logs — every cycle prints a `HEARTBEAT` line with a UTC
  timestamp. That is the cadence measurement.
