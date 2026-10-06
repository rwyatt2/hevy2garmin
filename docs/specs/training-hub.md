# Spec: personal training data hub

Status: draft for approval. No code exists. Claims checked on 2026-10-06.

## 1. Objective

Bring body weight, intake, recovery, and Hevy training data together in one place I own, so I can make four recurring decisions during my cut with less guesswork. The hub reads data and does not write it back. It must keep working without manual exports.

### Assumptions

- **A1.** I use an iPhone. This matters only for the Cronometer intake bridge in Phase 2.
- **A2.** The hub runs on my Mac, which is awake most mornings. Missed runs catch up the next time it wakes.
- **A3.** Cirqa is the Garmin CIRQA Smart Band, which syncs into Garmin Connect. "Garmin Connect via Cirqa" means the data comes from Garmin Connect, not from a separate Cirqa service.
- **A4.** My target rate of loss is not stated. The decision-1 thresholds below are proposed defaults for me to confirm (see Open questions).
- **A5.** I'm willing to log my morning weight in one app if that app has a supported API.

Status labels used below:

- **V** means verified, with a link or file path.
- **A** means assumed, not confirmed.

## 2. Decisions to metrics

| # | Decision | Cadence | Metric | Source | Field | Status |
|---|---|---|---|---|---|---|
| 1 | Is the weight trend on pace? Hold, raise, or lower calories from 2,500 | Weekly | Morning body weight, smoothed into a trend | Hevy body measurements (primary) | `GET /v1/body_measurements` → `weight_kg`, keyed by `date` | V [1][2] |
| 1 | | | Same (secondary) | Garmin Connect weigh-ins | `/weight-service/weight/range/{d}/{d}`, parsed to `WeightEntry.weight_grams` | V `web/node_modules/garmin-auth/dist/endpoints.js`, `health-parsers.d.ts` |
| 1 | | | Waist, weekly | Hevy body measurements | `waist` (cm) | V [2] |
| 1 | | | Daily intake kcal, to check adherence before changing the target | Cronometer → Apple Health → Health Auto Export | Apple Health "Dietary Energy", written by Cronometer | A (see 3.2) |
| 2 | Train hard, train light, or recover | Daily | Overnight HRV vs baseline | Garmin Connect | `/hrv-service/hrv/{date}` → `hrv_last_night_avg`, `hrv_weekly_avg`, `hrv_baseline`, `hrv_status` | V `garmin-auth/dist/health-parsers.d.ts` (Hrv) |
| 2 | | | Sleep duration and score | Garmin Connect | `/wellness-service/wellness/dailySleepData/{display}` → `total_sleep_seconds`, `sleep_score` | V same file (SleepDetail) |
| 2 | | | Resting HR vs 7-day average | Garmin Connect | `/usersummary-service/usersummary/daily/{display}` → `resting_heart_rate`, `rhr_7day_avg` | V same file (DailyHealth) |
| 2 | | | BJJ day | Config (Tue, Thu, Sat) | static schedule | V (your brief) |
| 3 | Is training volume holding in the deficit? | Weekly | Hard sets per muscle group, excluding warm-ups | Hevy workouts | `GET /v1/workouts/events?since=` → `exercises[].sets[]` (`type`, `weight_kg`, `reps`, `rpe`) plus `exercise_template_id` | V [2][3] |
| 3 | | | Volume load (kg × reps) and top-set estimated 1RM for 3 to 5 anchor lifts | Hevy workouts | same | V [3] |
| 3 | | | Muscle-group attribution | This repo | `aggregateMuscleVolumes` and `MuscleGroup` | V `typescript/src/muscle-groups.ts` (exported as `hevy2garmin/muscle-groups`) |
| 4 (optional) | Does right medial knee sensitivity track with specific exercises or volume? | Daily check-in | Knee score 0 to 10 | Manual (Raycast command) | `checkin.knee_score` | A (design choice) |
| 4 | | | Lower-body sets in the prior 48 h, by exercise | Hevy workouts | same as decision 3 | V |

### Excluded data

| Data | Why it's out |
|---|---|
| Garmin active calories and total calories (`active_kilocalories`, `total_kilocalories`) | Your rule: wrist-estimated burn never enters the food budget. These are never stored, which enforces the rule in the schema (section 5). |
| Steps | Doesn't serve any of the four decisions. Using steps to adjust food would break the active-calories rule by the back door. |
| Stress, Body Battery, training readiness | Garmin derives these from HRV, heart rate, and sleep. Adding them double-counts inputs already used in decision 2 and adds opaque scoring. Revisit only if decision 2 proves noisy. |
| Cronometer fluids, micronutrients, macros | Don't serve the four decisions. Protein is a reasonable future addition for a cut (see Open questions), but it isn't requested. |
| Garmin SpO2, respiration, skin temperature | Don't serve the four decisions. |
| Cronometer body weight | Cronometer has no supported read path (3.2). Weight comes from Hevy or Garmin instead. Cronometer already imports weight from Garmin [9], so logging weight in Garmin keeps Cronometer in sync with no extra entry. |

## 3. Source assessment

| Source | Official API for personal automated use | Auth | Rate limits | Terms | Unofficial options and break risk | Cadence needed vs available | What this repo already provides | Status |
|---|---|---|---|---|---|---|---|---|
| Hevy | Yes. Public REST API v1 at `api.hevyapp.com`, Hevy Pro only. Covers workouts, a workout change feed (`/v1/workouts/events?since=`), and body measurements keyed by date. | `api-key` header with a per-account key from hevy.com/settings?developer | Not published. No rate-limit headers and no 429 declared. The repo self-throttles at 1 s per call and retries 429 and 5xx with backoff. | No API-specific clause found in the general T&C. The spec itself warns: "we make no guarantees that we won't completely change the structure or abandon the project entirely". | Not needed. | Need: daily (weight) and after each workout. Available: on demand. The change feed covers workouts only, so measurements need a page read (10 per page, newest first). | `HevyClient` with retries and pacing (`typescript/src/hevy.ts`, npm `hevy2garmin/hevy`). The API key is already stored for this app (`HEVY_API_KEY` in `web/.env.example`, or `platform_credentials`). | V [1][2][3][4], `typescript/src/hevy.ts` |
| Cronometer | No. Cronometer staff say there is no public API. API access exists only for HIPAA practitioner and enterprise plans. | n/a | n/a | The ToS prohibits scraping and automated scripts "for AI training or related purposes". No general clause on personal automated access was confirmed. | `gocronometer` and personal MCP bridges replay your username and password against Cronometer's internal GWT web API. Values like `GWTPermutation` change with Cronometer releases. The library README claims a 10-export-per-day limit. Break risk: **high**. Account risk: **unknown**. | Need: daily intake, finalized next morning. Official: manual CSV export only, which breaks the no-manual-exports constraint. | Nothing. | V [5][6][7][8] |
| Cronometer → Apple Health → Health Auto Export (recommended bridge) | Indirect and supported by each vendor. Cronometer's iOS app writes nutrients to Apple Health. Health Auto Export runs automations that write JSON to iCloud Drive or POST to a REST endpoint. | Apple Health permissions on the phone. No Cronometer credentials. | n/a | Each app is used as designed. | Not scraping, but it depends on iOS background execution and a paid third-party app. Break risk: **medium**. | Need: daily. Available: whenever iOS runs the automation, which is opportunistic. | Nothing. | V [10][11][12]. A: that Cronometer writes "Dietary Energy" specifically (its blog lists "13 Different Nutrients"), that background runs are reliable enough, and that automations need a paid tier. |
| Garmin Connect (Cirqa data) | Business only. The Garmin Connect Developer Program (Health API) is for enterprise use, approved by application, with OAuth 2.0. Not available for a personal hub. | n/a for official. Unofficial: DI OAuth tokens obtained once by login with MFA, then refreshed. | Not published for the unofficial API. `garmin-auth` defaults to 1 s between calls and backs off on 429 (30 s base, 3 retries). | Garmin's consumer terms on automated access were not confirmed (the page didn't render for text checks). Treat this as a grey area. | `garmin-auth` (TS 0.7.0, same author as this repo) plus `garminconnect` (Python). These are the same mobile-app endpoints this repo already uses for HR and uploads. Break risk: **medium**. Garmin auth changes have already forced fixes (#459, #679). | Need: daily, after the morning sync. Available: once the band syncs to the phone app. | `GarminClient`, `FileTokenStore`, an endpoint catalogue (sleep, HRV, user summary, weigh-ins), and typed parsers (`parseSleep`, `parseHrv`, `parseDailyHealth`, `parseWeightEntries`). Also `getDailyHeartRate` in `typescript/src/garmin.ts`. | V [13][14][15], `web/node_modules/garmin-auth/dist/*.d.ts`, `typescript/src/garmin.ts`. A: ToS status. |
| CIRQA band coverage | n/a | n/a | n/a | n/a | n/a | n/a | n/a | V: the band records sleep stages, sleep score, HRV, and resting HR, and syncs to Garmin Connect [16][17]. A: that HRV status (balanced or unbalanced) is computed for CIRQA as it is for watches. |
| Manual (waist, knee) | Waist goes into the Hevy app's body measurements, so it arrives through the official API. Knee goes into a local Raycast command. | n/a | n/a | n/a | n/a | Waist weekly; knee daily, optional. | n/a | V [2] for waist; A for the Raycast design. |

### Plain statements

- **Cronometer can only be read directly by replaying your credentials against an undocumented API.** I recommend not doing that. Use the Apple Health bridge for intake (Phase 2), and take weight from Hevy or Garmin.
- If the Apple Health bridge proves unreliable, the fallback is **not integrating intake**. The weekly note then asks one yes/no question ("Averaged within 150 kcal of target? y/n") and makes the calorie call on weight trend plus that answer. That is a 5-second judgment, not an export.
- **Garmin has no personal API.** The hub uses the same unofficial path this repo already depends on. If it breaks, decision 2 degrades to "no data, decide by feel". Decisions 1 and 3 keep working.

## 4. Repository findings

Read before designing:

- `README.md` (line 13): the Python dashboard is gone as of 0.12.0. The dashboard is the Next.js app in `web/`. The PyPI CLI is deprecated, with no releases after **2026-10-31**.
- **`docs/AI_HANDOVER.md` is stale.** It describes a Python package (`src/hevy2garmin/*.py`) that uses SQLite and `garminconnect`. The current shape is:
  - a TypeScript engine published to npm as `hevy2garmin` 0.11.0 (`typescript/`)
  - `garmin-auth` 0.7.0 with DI tokens and an SSO Worker
  - the Next.js `web/` app on Postgres (Neon), with nine `CREATE TABLE IF NOT EXISTS` statements in `web/lib/schema.ts`
  - The Python sources still exist under `src/`.
- **`docs/CUTOVER.md` is partly stale.** It describes running the Python and web dashboards side by side and rolling back to `api/index.py`, but no `api/` directory exists in the tree.
- Deployment:
  - `web/.vercel/project.json` exists (gitignored), so `web/` is linked to a Vercel project.
  - `web/vercel.json` schedules `/api/cron/sync` daily at 08:00 UTC.
  - Garmin login goes through a Cloudflare Worker. The latest commit (#679) points it at the shared SSO Worker (`garmin-auth/sso-worker`, default `garmin-auth-sso.gkos.workers.dev`). The README still names `hevy2garmin-exchange-di.gkos.workers.dev`.
  - Hosting the hub doesn't depend on any of this. The recommended design runs locally and needs neither Vercel nor the Worker. **A:** a residential Mac can log in to Garmin directly, as the Python CLI did. The README says only cloud IPs are blocked.

## 5. Architecture

### 5.1 Where it lives (recommended: local-first, separate private project)

A new private repository, for example `~/Projects/training-hub`. It's a small Node/TypeScript CLI that depends on the published npm packages `hevy2garmin` (Hevy client, muscle groups) and `garmin-auth` (Garmin client, file token store, endpoints, parsers). It stores data in **SQLite** on the Mac.

Why:

- **Upstream merge cost: zero.** Nothing lands in the fork, so "Sync fork" never conflicts, and nothing reaches upstream.
- **Not built on the Python CLI**, which stops getting releases on 2026-10-31. It reuses the maintained TS packages through npm instead of copying code.
- **Data ownership.** Health data stays on a FileVault disk you control. No Neon or Vercel copy.
- **Upkeep.** One process, one file, launchd for scheduling. No server, no cron secret, no deploy.
- **SQLite over DuckDB.** The workload is small, frequent, idempotent upserts by a single writer, which SQLite handles natively. DuckDB is better at analytics this data doesn't need. A DuckDB query layer can be added later over the SQLite file if wanted.

Tradeoffs accepted:

- It runs only while the Mac is on. Missed schedules run on wake.
- No phone dashboard. The Obsidian note syncs wherever your vault syncs.
- Garmin tokens need a one-time interactive login with MFA on the Mac.

### 5.2 Rejected alternative: inside this fork's `web/` app on Vercel and Neon

Rejected because:

- Every new table, route, and page lives in files upstream also edits, so each "Sync fork" risks conflicts in `web/lib/schema.ts`, `proxy.ts`, and the nav.
- Health data would live in a third-party cloud database, against your local-first preference.
- The web path has known database bootstrapping and cron-auth gaps (#475, #473 in `docs/CUTOVER.md`).
- The Apple Health intake bridge would need a public HTTPS endpoint instead of a local folder.

What you'd gain: a phone-reachable dashboard and an existing Garmin token in `platform_credentials`. Neither is needed for the four decisions.

"A separate app sharing the Neon database" was also considered. It removes the merge cost but keeps the cloud custody and adds a second deploy. It is the worst of both options.

### 5.3 Data model

Canonical units are stored (kg, cm, seconds, kcal). Display units are lb and in. Dates are local calendar dates in your timezone.

```sql
-- One row per day, per metric, per source. Re-ingesting overwrites the same row.
CREATE TABLE daily_metric (
  date        TEXT NOT NULL,          -- YYYY-MM-DD, local
  metric      TEXT NOT NULL,          -- must be in metric_def (allowlist)
  source      TEXT NOT NULL,          -- 'hevy' | 'garmin' | 'apple_health' | 'manual'
  value       REAL,
  fetched_at  TEXT NOT NULL,
  PRIMARY KEY (date, metric, source)
);

-- The allowlist. Active calories are intentionally absent, so they cannot be stored.
CREATE TABLE metric_def (
  metric   TEXT PRIMARY KEY,          -- weight_kg, waist_cm, intake_kcal, sleep_s, sleep_score,
                                      -- hrv_last_night_ms, hrv_weekly_avg_ms, hrv_baseline_low_ms,
                                      -- hrv_status, rhr_bpm, rhr_7d_bpm, knee_score
  unit     TEXT NOT NULL,
  decision INTEGER NOT NULL           -- which decision it serves (1-4)
);

-- Which source wins when two report the same metric on the same day. Lowest rank wins.
CREATE TABLE source_precedence (
  metric TEXT NOT NULL, source TEXT NOT NULL, rank INTEGER NOT NULL,
  PRIMARY KEY (metric, source)
);
-- weight_kg: hevy=1, garmin=2     (deliberate morning log beats incidental scale or manual entry)
-- waist_cm:  hevy=1
-- intake_kcal: apple_health=1     (Cronometer is the origin)
-- recovery metrics: garmin=1

CREATE VIEW daily_resolved AS        -- one value per (date, metric) after precedence
  SELECT date, metric, value, source FROM (
    SELECT m.*, ROW_NUMBER() OVER (PARTITION BY m.date, m.metric ORDER BY p.rank) AS rn
    FROM daily_metric m JOIN source_precedence p USING (metric, source)
    WHERE m.value IS NOT NULL
  ) WHERE rn = 1;

-- Training detail for decisions 3 and 4. Upserted from the Hevy change feed.
CREATE TABLE workout     (hevy_id TEXT PRIMARY KEY, start_time TEXT, end_time TEXT, title TEXT, updated_at TEXT, deleted INTEGER DEFAULT 0);
CREATE TABLE workout_set (hevy_id TEXT, exercise_index INTEGER, set_index INTEGER,
                          exercise_template_id TEXT, exercise_title TEXT, set_type TEXT,
                          weight_kg REAL, reps INTEGER, rpe REAL,
                          PRIMARY KEY (hevy_id, exercise_index, set_index));

-- Raw payloads so parsers can be fixed and re-run without re-fetching.
CREATE TABLE raw_payload (source TEXT, key TEXT, fetched_at TEXT, json TEXT, PRIMARY KEY (source, key));

-- Ingestion bookkeeping and alerting.
CREATE TABLE ingest_run  (id INTEGER PRIMARY KEY, source TEXT, started_at TEXT, finished_at TEXT,
                          status TEXT, rows_upserted INTEGER, error TEXT);
CREATE TABLE cursor      (source TEXT PRIMARY KEY, value TEXT);   -- e.g. hevy events `since`

-- Phase 5 only.
CREATE TABLE checkin (date TEXT PRIMARY KEY, knee_score INTEGER CHECK (knee_score BETWEEN 0 AND 10), note TEXT);
```

How the active-calories rule is enforced:

- Ingestion writes only metrics present in `metric_def`.
- A unit test fails if any `*_kilocalories` key from Garmin reaches `daily_metric`.
- `intake_kcal` is the only kcal metric. The calorie target is computed from the 2,500 baseline and weight trend only.

### 5.4 Ingestion

| Concern | Design |
|---|---|
| Scheduling | Two launchd jobs. 07:30 daily: Garmin (yesterday and today), Hevy measurements, Hevy workout events, Apple Health folder. 21:30 daily: Hevy workout events. **A:** launchd runs a missed calendar job once on wake. |
| Idempotency | Every write is an upsert on a natural key: `(date, metric, source)`, `hevy_id`, `(hevy_id, exercise_index, set_index)`. Re-running any day is safe. Hevy `deleted` events mark the workout deleted. |
| Incremental reads | Hevy workouts: `/v1/workouts/events?since=<cursor>`, then advance the cursor after a successful commit. Hevy measurements: read pages until a date older than the last stored date. Garmin: always re-pull a 3-day window, because late band syncs fill in yesterday. Apple Health: process new JSON files from the Health Auto Export iCloud Drive folder and move them to `processed/`. |
| Backfill | `hub backfill --source garmin --from 2026-08-01`, one day at a time at 1 s pacing. Hevy: events `since=1970-01-01` once. Default window: from the start of the cut. |
| Failure alerts | Each run writes an `ingest_run` row. On failure, show a macOS notification via `osascript` with source and error. The weekly note and the Raycast command print a freshness line ("Garmin: last good 2 days ago"). Garmin `GarminAuthenticationError` gets one message: "Run `hub garmin-login`". |
| Rate limits | Reuse `HevyClient` pacing (1 s) and `garmin-auth` `RateLimiter` defaults. A daily run makes about 10 Garmin calls and 2 to 4 Hevy calls. |

### 5.5 Where I see it (recommended: weekly Obsidian note, plus one Raycast command)

- **Weekly note**, written Sunday at 19:00 to `<vault>/Training/2026-W41.md`. It answers decision 1, and decision 3 from Phase 4. It's a plain Markdown table with a one-line recommendation, for example: "Hold 2,500. Trend −1.6 lb/wk, inside the 1.2 to 2.4 band." This is the smallest surface for a weekly decision. No server, no UI code.
- **Raycast script command `Train today?`** prints one line for decision 2, for example: "Light. HRV below baseline, BJJ tonight." Raycast script commands are plain shell scripts [18]. The same command takes `knee 3` for decision 4.
- A dashboard is rejected for now. It's the largest build and serves no decision better than a note.

### 5.6 Security

- **Data:** `~/Library/Application Support/training-hub/hub.sqlite`, mode `600`, on FileVault. It's in Time Machine backups; that is acceptable. It's never in a git repository.
- **Hevy API key:** macOS Keychain (`security add-generic-password -s training-hub-hevy`), read at runtime. Not in `.env` or shell history.
- **Garmin tokens:** `garmin-auth` `FileTokenStore` in `~/Library/Application Support/training-hub/garmin/` (`garmin_tokens.json`), directory mode `700`. **A:** the file is plaintext JSON. Wrap it in Keychain in a later phase if wanted. Your Garmin password is typed once into the interactive `hub garmin-login` and never stored.
- **Cronometer:** no credentials anywhere.
- **Apple Health files:** land in iCloud Drive (encrypted at rest by Apple, readable by your Apple ID), and are deleted after import.
- **Obsidian note:** holds aggregates only (weekly averages and the call), not raw daily health rows. If the vault syncs through a third party, that party can read the note.
- **Who can read it:** your macOS user account. No network listener, so no remote reader.

## 6. Decision logic (proposed defaults, confirm before build)

1. **Weight pace.**
   - Trend = 7-day exponentially weighted mean of morning weight.
   - Pace = slope of the trend over the last 14 days, in lb/week.
   - Target band, proposed: 0.5 to 1.0% of current trend weight per week, about 1.2 to 2.4 lb/week at 240 lb.
   - Need at least 5 weigh-ins in each of the last 2 weeks, or the call is "no call, log more".
   - Losing slower than the band for 2 consecutive weeks: lower by 150 kcal, but only if intake adherence passes. Adherence = at least 6 logged days averaging within 150 kcal of target. If adherence fails, the call is "fix adherence first".
   - Losing faster than the band: raise by 150 kcal.
   - Otherwise: hold.
   - The calorie target never uses Garmin calories.
2. **Train today.**
   - Recover: `hrv_status` is LOW or POOR, or last night's HRV is below the baseline low, or RHR is more than 5 bpm above its 7-day average.
   - Light: sleep under 6.5 h, or `hrv_status` is UNBALANCED, or it's a BJJ day.
   - Hard: otherwise.
3. **Volume holding.**
   - Weekly hard sets per muscle group (normal and failure sets, not warm-ups, weighted 1.0 primary and 0.33 secondary per `typescript/src/muscle-groups.ts`) compared with the 4-week pre-cut baseline.
   - Volume holds if sets are at or above 90% of baseline and anchor-lift top-set e1RM is at or above 95%.
4. **Knee (optional).** A table of knee score against lower-body exercises done in the prior 48 h, shown after 4 or more weeks of check-ins. It describes patterns only; it makes no causal claim.

## 7. Phased plan

| Phase | Scope | Decision | Done when |
|---|---|---|---|
| **1** | SQLite schema, Hevy key in Keychain, Hevy body-measurements ingest (weight and waist), launchd 07:30, `ingest_run` and notifications, weekly Obsidian note with trend, pace, and hold/raise/lower call. You log morning weight and weekly waist in Hevy. | 1 only | 2 consecutive Sunday notes generated with no manual step. The note's numbers match Hevy's history. |
| 2 | Intake adherence: Health Auto Export → iCloud Drive JSON → `intake_kcal`. The adherence gate is added to the decision-1 call. Fallback if unreliable: the y/n adherence question. | 1 | 14 consecutive days with intake present by 07:30 next day. |
| 3 | Garmin: `hub garmin-login`, sleep, HRV, and RHR ingest with a 3-day re-pull, Raycast `Train today?`. Garmin weigh-ins added as the secondary weight source. | 2 | 7 days of daily calls. Auth failures surface as one notification. |
| 4 | Hevy workouts change feed, `workout` and `workout_set` tables, weekly sets per group and anchor e1RM in the note. | 3 | Weekly volume matches a hand count for one week. |
| 5 (optional) | Knee check-in via Raycast and the 48 h lag table. | 4 | 28 check-ins logged. |

## 8. Risks

| Risk | Likelihood | Effect | Mitigation |
|---|---|---|---|
| Garmin changes auth or endpoints | Medium | Decision 2 goes dark | Isolated to Phase 3. Raw payloads kept. Decisions 1 and 3 don't depend on Garmin. Track `garmin-auth` releases. |
| Hevy changes or abandons API v1 (its own spec warns of this) | Low to medium | Decisions 1 and 3 go dark | Raw payloads kept. Garmin weigh-ins as fallback for weight. Hevy CSV export as a manual last resort. |
| iOS doesn't run Health Auto Export on schedule | Medium | Intake missing or late | Freshness line in the note. Fallback y/n adherence question. |
| Cronometer doesn't write Dietary Energy to Apple Health | Low to medium (unverified) | Phase 2 blocked | Check on the phone before Phase 2. Fallback as above. Do not fall back to credential replay. |
| Mac off for days | Low | Stale note | Catch-up on wake. Freshness line. |
| Two weight sources disagree (scale vs manual) | Medium | Noisy trend | Precedence table. Only one source feeds the trend. |
| Wrist HRV on CIRQA is noisy | Medium | Bad daily calls | Use HRV status and the baseline band, not single-night values. Light rather than skip. |
| Health data leaks via a synced vault | Low | Privacy | Aggregates only in notes. Raw data stays in SQLite. |
| Knee correlation is spurious | High | False conclusions | Phase 5 labelled descriptive only. Requires 4 or more weeks of data. |

## 9. Open questions

1. What weekly loss rate do you want: the proposed 0.5 to 1.0% of bodyweight, a fixed lb/week, or a date to reach 215 lb?
2. Where will you log morning weight: Hevy (official API, recommended for Phase 1), Garmin Connect (one entry that also flows into Cronometer [9], but unofficial API), or both?
3. Do you use an iPhone, and are you willing to pay for Health Auto Export automations? If not, Phase 2 uses the y/n fallback.
4. What step size for calorie changes: 150 kcal as proposed, 100, or 250? Should there be a floor below which the hub won't recommend going?
5. What sleep and HRV thresholds feel right for "light" vs "recover"? Should BJJ days always mean light lifting, or only after a poor night?
6. Which 3 to 5 anchor lifts define "volume holding", and which 4 weeks are the pre-cut baseline?
7. Where is the Obsidian vault, and does it sync through a third party?
8. Should protein intake be added to decision 1 or 3 (relevant to keeping muscle during a cut)? It isn't in scope today.
9. Is the Garmin login Worker and Vercel deployment for `web/` staying as-is? The hub doesn't need either, but they affect whether `web/` remains your sync path after 2026-10-31.

## 10. Success measures

- You open the weekly note at least 3 Sundays in every 4.
- It changes at least one calorie decision a month (a raise, a lower, or "fix adherence first").
- Zero manual exports. Intake and weight arrive on their own.
- Weight is present on at least 6 of 7 days. Garmin recovery data is present on at least 90% of days once Phase 3 ships.
- Every ingest failure appears as a notification within 24 hours.
- Upkeep is under 30 minutes a month after Phase 3.

## References

1. Hevy API key and Pro requirement: https://docs.rs/hevy/latest/hevy/ and https://raw.githubusercontent.com/api-evangelist/hevy/refs/heads/main/authentication/hevy-authentication.yml
2. Hevy body measurements (fields incl. `weight_kg`, `waist`; keyed by date): https://github.com/diecoscai/hevy-mcp/blob/main/docs/tools.md and https://apis.io/apis/hevy/hevy-measurements-api/
3. Hevy conventions (change feed, pagination, no published rate limit, "abandon the project" warning): https://raw.githubusercontent.com/api-evangelist/hevy/refs/heads/main/conventions/hevy-conventions.yml
4. Hevy official docs: https://api.hevyapp.com/docs/ and terms: https://www.hevyapp.com/legal/terms-and-conditions/
5. Cronometer has no public API (staff reply): https://forums.cronometer.com/discussion/1801/cronometer-export-api
6. Cronometer third-party API request thread: https://forums.cronometer.com/discussion/3809/does-cronometer-offer-an-api-for-third-party-applications-to-retreive-user-data-from
7. Cronometer Terms of Service: https://cronometer.com/terms/
8. `gocronometer` (GWT replay, changing values, 10 exports/day claim): https://github.com/jrmycanady/gocronometer
9. Cronometer imports weight, sleep, HRV, and RHR from Garmin: https://support.cronometer.com/hc/en-us/articles/16957825757204-Garmin and https://cronometer.com/blog/garmin/
10. Cronometer exports nutrients to Apple Health: https://cronometer.com/blog/apple-health/
11. Health Auto Export automations (iCloud Drive, REST): https://help.healthyapps.dev/en/health-auto-export/automations
12. Health Auto Export REST automation: https://help.healthyapps.dev/en/health-auto-export/automations/rest-api
13. Garmin Health API is for business use: https://developer.garmin.com/gc-developer-program/health-api
14. Garmin Developer Program FAQ (enterprise only, OAuth 2.0): https://developer.garmin.com/gc-developer-program/program-faq
15. `python-garminconnect` (HRV, sleep, body composition methods): https://github.com/cyberjunky/python-garminconnect
16. CIRQA announcement (sleep stages, score, HRV): https://www.garmin.com/en-US/newsroom/press-release/wearables-health/meet-cirqa-smart-band-the-screen-free-health-and-fitness-tracker-from-garmin/
17. CIRQA manual (syncs with Garmin Connect): https://www8.garmin.com/manuals/webhelp/GUID-D3BE589F-1A6C-4B6F-9ABC-07DB9AA6C739/EN-US/GUID-4BAFDB96-1102-466B-AFC6-C9AF7E59059F.html
18. Raycast script commands: https://github.com/raycast/script-commands
