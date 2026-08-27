# cardio-minutes.csv — Setup

Pipeline that pulls per-sample heart-rate data from Garmin Connect and
maintains one always-sorted `cardio-minutes.csv` in this repo.

## What goes where

```
logtrim/
├── cardio-minutes-pipeline/
│   ├── cardio_minutes.py      # the pipeline
│   ├── garmin_login.py        # one-time token generator
│   └── CARDIO-MINUTES-SETUP.md
├── .github/workflows/
│   └── cardio-minutes.yml     # daily scheduled run (off until you opt in)
└── cardio-minutes.csv         # created on first run
```

All of these files ship with the repo — if you forked LogTrim, you
already have them. The workflow does nothing until you complete the
setup below, so forks that don't use Garmin never see it run.

## One-time setup (~5 minutes)

1. **Generate the Garmin token blob** on your computer:
   ```
   pip install garminconnect
   python cardio-minutes-pipeline/garmin_login.py
   ```
   Enter your Garmin email/password (MFA code if prompted). It prints a
   long base64 string. Your password is used once and never stored;
   the token is what the Action uses, and it lasts about a year.

2. **Add the secret:** GitHub → YOUR-USERNAME/logtrim → Settings →
   Secrets and variables → Actions → Secrets tab → New repository secret.
   Name: `GARMIN_TOKENS`. Value: the base64 string from step 1.
   (This is the same secret the Garmin Sync daily-stats workflow uses —
   set it once and both pipelines share it.)

3. **Turn the workflow on:** same page, Variables tab → New repository
   variable. Name: `GARMIN_ENABLED`. Value: `true`.

4. **First run:** Actions tab → `cardio-minutes` → Run workflow.
   It will fetch your 30 most recent activities and commit the CSV.
   After that it runs daily at ~5:20 AM ET automatically.

## The file

One row per clock minute (America/New_York) of any Garmin activity with HR:

| column | meaning |
|---|---|
| `date` | NY calendar date |
| `activity_id` | Garmin activity ID (dedup key) |
| `activity_type` | e.g. `running`, `treadmill_running`, `elliptical` |
| `minute` | `YYYY-MM-DD HH:MM`, NY local |
| `avg_hr_weighted` | time-weighted mean HR for that minute |
| `seconds_covered` | seconds of actual sensor coverage in that minute |

Design notes baked in: readings are forward-filled to the next sample
(smart-recording semantics), gaps are capped at 30 s, boundary-spanning
readings are split proportionally, every minute is written regardless of
coverage (apply your ≥10 s filter at analysis time), no zone columns —
classify against whatever zones you like, re-rollable forever.

## Backfill

The daily run fetches the 30 most recent activities. For a deeper
historical backfill, run once with a bigger window (locally or by
editing the workflow env):

```
ACTIVITY_FETCH_LIMIT=300 GARMIN_TOKENS="..." python cardio-minutes-pipeline/cardio_minutes.py
```

## When auth eventually expires (~1 year)

The Action will start failing with an auth error. Re-run
`garmin_login.py` and update the `GARMIN_TOKENS` secret. That's it.

## Note on the Garmin API

This uses Garmin's unofficial Connect API via the `garminconnect`
library. It's well-maintained but not contractual — if Garmin changes
something, the Action may fail until the library updates. The CSV is
append-only from your side, so nothing already written is ever at risk.
