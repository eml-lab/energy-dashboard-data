# MISO Data Archive → Energy Dashboard

The `miso-data-archive` project archives MISO **Actual Load** and the
**Medium-Term Load Forecast (MTLF)** by Local Resource Zone (LRZ) from the MISO
Data Exchange API. Every morning it exports cleaned parquet files to the shared
repository
[eml-lab/energy-dashboard-data](https://github.com/eml-lab/energy-dashboard-data),
with **one folder per archive day**:

```
energy-dashboard-data/
├── README.md                          ← this file
└── miso/
    ├── publish_date=2026-10-10/
    │   ├── actual_load.parquet        ← actual load for 2026-10-09 (D-1)
    │   └── mtlf_day_ahead.parquet     ← day-ahead MTLF for 2026-10-10 (D), from init 2026-10-09
    └── publish_date=2026-10-11/
        ├── actual_load.parquet        ← actual load for 2026-10-10 (D-1)
        └── mtlf_day_ahead.parquet     ← day-ahead MTLF for 2026-10-11 (D), from init 2026-10-10
```

A new `publish_date=D` folder is added every morning, and older folders are
left as they are.

> **This README is published automatically.** The export copies it from the
> `miso-data-archive` project to the root of the shared repository. Edit it in
> the project, not on GitHub, or the next export will overwrite your change.

The project's original README, `README_OG.md` (in the `miso-data-archive`
project), covers the archive itself: the API key, backfill modes and the
"never overwrite" storage design.

---

## Daily schedule

All times are US Eastern clock time, which is the archiving Mac's local time.
Let **day D** be the day the jobs run, which is the archive day.

| Time (ET) | What happens | Data involved |
|---|---|---|
| ~2:00 AM | MISO publishes actual load for the previous day | operating day **D-1** |
| **3:00 AM** | **Archive: actual load** (`run_actual_load.py`) | D-1, HE1 → HE24 |
| ~7:00 AM | MISO makes yesterday's MTLF run queryable in the API | init **D-1** |
| **8:00 AM** | **Archive: MTLF** (`run_mtlf.py`) | init D-1, the full 7-day horizon D … D+6 |
| **8:30 AM** | **Export to GitHub** (`run_export.py`) | new folder `miso/publish_date=D/` with both files, pushed **together** |

- The 8:30 export first checks that both of this morning's archive runs have
  landed. If MISO is slow and the MTLF job is still retrying, the export waits
  for it, checking every minute for up to 45 minutes, so both files arrive in
  the same push. If the MTLF data still hasn't arrived after that, the export
  pushes `actual_load.parquet` alone and logs a warning. The next run adds the
  missing `mtlf_day_ahead.parquet` to that folder.
- These jobs are macOS `launchd` jobs (`edu.lab.miso.actualload`,
  `edu.lab.miso.mtlf`, `edu.lab.miso.export`). They run in the background
  while you are logged in. If the Mac is asleep at the scheduled time, the job
  runs when it wakes. All three also run once at login/boot, so nothing is lost
  if the Mac was off. Details are in `scheduling/AUTOMATION.md` (in the
  project).
- The schedule follows the Mac's clock through EST/EDT changes, as MISO's
  publish times do. The **data** is always in fixed EST (UTC−05:00), so every
  `TargetDate` has exactly 24 hours, with no missing or doubled hour at
  daylight-saving changes.

---

## Timing logic: why the "day-ahead" forecast is for day D

MISO creates an MTLF run every day. The run created on **init date I** forecasts
target days I+1 through I+7, and its **day-ahead** forecast is the one for
**I+1**.

The catch is a publication lag. The API only serves the run for init I from
about **7:00 AM on day I+1**. So on day D the newest run available is
**init D-1**, and its day-ahead forecast is for **day D**, the day that has
already started.

```
        day D-1                     │                 day D
  ──────────────────────────────────┼──────────────────────────────────────────
  MISO creates MTLF run init D-1    │ 02:00  MISO publishes actual load for D-1
  (forecasts D, D+1, … D+6)         │ 03:00  archive actual load D-1
                                    │ 07:00  init D-1 becomes queryable (API lag)
                                    │ 08:00  archive MTLF init D-1
                                    │ 08:30  push → miso/publish_date=D/
                                    │          actual_load.parquet    = D-1
                                    │          mtlf_day_ahead.parquet = D
```

So **when the data reaches GitHub on day D, the "day-ahead forecast" is for
day D, NOT day D+1.** This is why the two files in one folder cover different
dates:

| File in `miso/publish_date=D/` | `TargetDate` inside | Source |
|---|---|---|
| `actual_load.parquet` | **D-1** | actual load archived at 3:00 AM on D |
| `mtlf_day_ahead.parquet` | **D** | init D-1's day-ahead forecast, archived at 8:00 AM on D |

The general rule: **the forecast for `TargetDate` T always comes from init T-1**,
a lead of one day. It sits in folder `publish_date=T`, while the realized load
for T arrives a day later in folder `publish_date=T+1`.

**Example.** Folder `miso/publish_date=2026-10-11/`, pushed on Sunday
2026-10-11, contains actual load for 2026-10-10 and the MTLF day-ahead forecast
for 2026-10-11 from init 2026-10-10.

### Need a forecast for day D+1?

Then you must use the **second-day-ahead** forecast from **init D-1**, that is
`forecast_date = init + 2 days`. It is **not** in the exported files, which
hold only the day-ahead vintage. Every vintage is kept in the project's local
archive:

```python
import pandas as pd

# On D = 2026-10-11: init D-1 = 2026-10-10, so D+1 = 2026-10-12 is its second-day-ahead.
run = pd.read_parquet("data/mtlf/publish_date=2026-10-10/data.parquet")
d_plus_1 = run[run["forecast_date"] == "2026-10-12"]      # zones Z1..Z10, native fields
```

Note that the archive's `data/mtlf/publish_date=I` folders are named after the
**init** date, while the export's `miso/publish_date=D` folders are named after
the **archive day**. For MTLF, D = I + 1.

---

## Exported data structure

Every `miso/publish_date=D/` folder holds two parquet files. Each file covers
**one day** in long format: one row per **HE × zone**, so 24 × 7 = **168 rows**.
The data starts at `publish_date=2026-01-02`, the first day the archive has
both datasets for.

| Column (left → right) | Type | Meaning |
|---|---|---|
| `TargetDate` | string `YYYY-MM-DD` | The operating day being measured/forecast (fixed EST) |
| `HE` | int64, 1–24 | Hour ending (HE1 = 00:00–01:00 EST, HE24 = 23:00–24:00) |
| `LocalResourceZone` | string | `LRZ1`, `LRZ2_7`, `LRZ3_5`, `LRZ4`, `LRZ6`, `LRZ8_9_10`, `MISO` |
| `Realized` *(actual_load.parquet)* | float64 | Actual load for that hour, MW (hourly average; numerically equal to MWh) |
| `MTLF` *(mtlf_day_ahead.parquet)* | float64 | Day-ahead load forecast for that hour, MW |

Rows are sorted by `HE`, then zone, in the order listed above.

### `actual_load.parquet` (TargetDate = D-1)

MISO already reports actual load in the six LRZ groups. Those values are passed
through **unchanged**, and the `MISO` row is the sum of the six.

```
TargetDate  HE LocalResourceZone  Realized
2026-10-10   1              LRZ1    9747.1
2026-10-10   1            LRZ2_7   14550.9
2026-10-10   1            LRZ3_5    9470.7
2026-10-10   1              LRZ4    4226.7
2026-10-10   1              LRZ6    8441.9
2026-10-10   1         LRZ8_9_10   19315.7
2026-10-10   1              MISO   65753.0
```
*(first rows of `miso/publish_date=2026-10-11/actual_load.parquet`)*

### `mtlf_day_ahead.parquet` (TargetDate = D)

MISO publishes MTLF for ten individual zones (Z1–Z10). The export:

1. keeps only the **day-ahead** vintage, where `forecast_date` = init + 1 day,
2. sums the zones into the same groups used by the actual load, and into MISO:

| `LocalResourceZone` | Sum of MTLF zones |
|---|---|
| `LRZ1` | Z1 |
| `LRZ2_7` | Z2 + Z7 |
| `LRZ3_5` | Z3 + Z5 |
| `LRZ4` | Z4 |
| `LRZ6` | Z6 |
| `LRZ8_9_10` | Z8 + Z9 + Z10 |
| `MISO` | Z1 + Z2 + … + Z10 |

```
TargetDate  HE LocalResourceZone     MTLF
2026-10-11   1              LRZ1  10133.0
2026-10-11   1            LRZ2_7  14089.0
2026-10-11   1            LRZ3_5   9118.0
2026-10-11   1              LRZ4   3889.0
2026-10-11   1              LRZ6   8112.0
2026-10-11   1         LRZ8_9_10  18367.0
2026-10-11   1              MISO  63708.0
```
*(first rows of `miso/publish_date=2026-10-11/mtlf_day_ahead.parquet`)*

### Cleaning rules

- A group or MISO total is only written for an hour when **every member zone
  is present**. A missing zone therefore leaves a gap instead of a quietly
  understated total, and the export logs it.
- Rows with a missing value or an invalid hour are dropped (logged).
  Duplicates are removed.
- Sums are rounded to 0.1 to remove floating-point noise. MISO's own values
  never have more than one decimal.
- If MISO ever adds a zone that isn't in the table above, the export logs an
  error and leaves it out. Update `LRZ_GROUPS` in `src/dashboard_export.py`.

### Using the files

The newest day straight from GitHub:

```python
import pandas as pd

base = "https://raw.githubusercontent.com/eml-lab/energy-dashboard-data/main/miso"
day = "2026-10-11"                                                   # publish_date D
actual = pd.read_parquet(f"{base}/publish_date={day}/actual_load.parquet")     # TargetDate D-1
mtlf = pd.read_parquet(f"{base}/publish_date={day}/mtlf_day_ahead.parquet")    # TargetDate D
```

All days, from a local clone of the shared repository
(`git clone https://github.com/eml-lab/energy-dashboard-data.git`):

```python
import glob
import pandas as pd

def read_all(name):
    files = sorted(glob.glob(f"energy-dashboard-data/miso/publish_date=*/{name}"))
    return pd.concat((pd.read_parquet(f) for f in files), ignore_index=True)

actual = read_all("actual_load.parquet")
mtlf = read_all("mtlf_day_ahead.parquet")

# Forecast vs realized for one zone (TargetDate, HE, MTLF, Realized).
# Only days present in both are kept, so the newest MTLF day (no realized yet) drops out.
zone = "LRZ6"
table = mtlf[mtlf.LocalResourceZone == zone].merge(
    actual[actual.LocalResourceZone == zone], on=["TargetDate", "HE", "LocalResourceZone"]
)[["TargetDate", "HE", "MTLF", "Realized"]]
```

Read each file name separately, as above, rather than reading the whole `miso/`
folder at once. The two files in a folder have different value columns
(`Realized` vs `MTLF`).

The export was checked against `MISO_MTLF_Realized_Example.csv`. Rebuilding
LRZ6 for 2026-07-01 → 2026-10-10 reproduces all 2,448 rows exactly, including
the dtypes.

**Layout history.** On 2026-10-11 the first export pushed two cumulative files,
`miso/actual_load.parquet` and `miso/mtlf_day_ahead.parquet`. They were
replaced the same day by the daily `publish_date=D` folders, which hold
identical values, and remain only in the git history.

---

## Running the export (in the `miso-data-archive` project)

```bash
venv/bin/python run_export.py              # what the 8:30 job runs: waits for this morning's data, then pushes
venv/bin/python run_export.py --no-wait    # push now with whatever is archived
venv/bin/python run_export.py --no-push    # only build the folders into exports/ (GitHub untouched)
```

Each run rebuilds every `publish_date` folder from the archive into `exports/`,
then commits and pushes **only the files that changed**. Normally that's the
new day's folder, plus `README.md` if it was edited. Unchanged days produce
byte-identical files, so they never create commits.

### Did it work?

`logs/export.log` ends with either of these, followed by `exit=0`:

- `Committed 2 changed file(s): MISO publish_date=2026-10-12: actual load 2026-10-11, MTLF day-ahead 2026-10-12`
  and then `Pushed to https://github.com/eml-lab/energy-dashboard-data.git`
- `Dashboard repository already up to date; nothing to push.`

`launchctl list | grep edu.lab.miso` shows the last exit code in the middle
column (`0` = success).

| Message in `logs/export.log` | What to do |
|---|---|
| `GitHub did not accept credentials` | Renew the token (setup steps 2–3 below) |
| `Permission to eml-lab/energy-dashboard-data.git denied` | That account lacks write access (setup step 1) |
| `Still missing MTLF init … after waiting 45 min` | MISO was late. The actual load was pushed, and the next run adds the MTLF file |
| `publish_date=…: no mtlf_day_ahead.parquet` / `no actual_load.parquet` | That day isn't archived yet (or is missing from the archive) |
| `has uncommitted changes to files this pipeline does not manage` | Someone edited files inside `dashboard_repo/`. Commit those edits from there or discard them |
| `Nothing to export` | The local archive (`data/`) is missing. Nothing was pushed |

### How the push works

- The export keeps its own clone of the shared repo in `dashboard_repo/`. Each
  run fetches GitHub's latest state, resets the clone to it, copies in the
  files, and commits and pushes **only what changed**.
- The export writes only `miso/publish_date=*/actual_load.parquet`,
  `miso/publish_date=*/mtlf_day_ahead.parquet` and the root `README.md`.
  Anything collaborators add elsewhere is picked up first and left alone. If
  someone pushes at the same moment, the export re-syncs and retries.
- `dashboard_repo/` is managed by the pipeline. Don't edit files in it. To
  change something else in the shared repository, use your own clone.
- Commits are authored as *MISO data archive (automated)* with a placeholder
  email. The repository is public, so commit emails are visible to anyone.
  You can change both in `.env`.
- Each new folder is about 8 KB, so the repository grows by roughly 3 MB a
  year.

### Export settings (`.env`)

All settings are optional; the defaults are shown.

| Setting | Default | Purpose |
|---|---|---|
| `DASHBOARD_REPO_URL` | `https://github.com/eml-lab/energy-dashboard-data.git` | Where to push |
| `DASHBOARD_REPO_BRANCH` | `main` | Branch to push to |
| `DASHBOARD_REPO_SUBDIR` | `miso` | Folder in the repo that holds the `publish_date=` folders (empty = repo root) |
| `DASHBOARD_REPO_DIR` | `./dashboard_repo` | Local clone managed by the export |
| `DASHBOARD_GIT_NAME` / `DASHBOARD_GIT_EMAIL` | `MISO data archive (automated)` / `miso-data-archive@noreply.invalid` | Commit author |
| `EXPORT_WAIT_MAX_MINUTES` / `EXPORT_WAIT_POLL_SECONDS` | `45` / `60` | How long the 8:30 run waits for the morning's archive runs |
| `EXPORT_START_DATE` | *(all history)* | Only export `TargetDate`s on/after this date |
| `GIT_EXECUTABLE` | `/usr/bin/git` | Git used for the push (Apple's git keeps the token in the keychain) |

---

## One-time GitHub setup

This is already done on the archiving Mac. Repeat steps 2–3 when the token
expires, and all steps on a new machine. The scheduled job pushes to GitHub
without anyone at the keyboard, so it needs a saved GitHub login. Run the
commands in the macOS **Terminal** app (Applications → Utilities → Terminal).

**0. Dry run (no GitHub).** Build the folders locally first:

```bash
cd ~/Projects/EML/miso-data-archive
venv/bin/python run_export.py --no-push
```

It should end with `exit=0`, and the `publish_date=…` folders will be in `exports/`.

**1. Choose the GitHub account that will push.** The repo belongs to the
`eml-lab` GitHub account.

- **A. A personal account.** On github.com, signed in as `eml-lab`, open the
  repo → **Settings → Collaborators → Add people** → the personal account.
  Then, signed in as that account, accept the invitation from the email or at
  <https://github.com/eml-lab/energy-dashboard-data/invitations>.
- **B. The `eml-lab` account itself.** Add this line to the end of `.env`
  (`open -e .env` opens it in TextEdit) before step 3:

  ```
  DASHBOARD_REPO_URL=https://eml-lab@github.com/eml-lab/energy-dashboard-data.git
  ```

  Having `eml-lab@` in the URL makes git ask for that account's token, even
  when another GitHub login is saved on the Mac.

**2. Create a personal access token**, a password meant for scripts. Sign in
to github.com as the account you chose, click the profile picture →
**Settings** → **Developer settings** → **Personal access tokens**:

- *A (personal account):* **Tokens (classic) → Generate new token (classic)**.
  Tick only **`public_repo`**. Fine-grained tokens can't push to a repository
  owned by another personal account.
- *B (`eml-lab`):* **Fine-grained tokens → Generate new token**. Under
  *Repository access* choose **Only select repositories →
  energy-dashboard-data**. Under *Permissions → Repository permissions* set
  **Contents: Read and write**.

Choose an expiration date and note it, because pushes stop when the token
expires. Copy the token right away; GitHub shows it only once.

**3. Push once from Terminal** so the token gets saved:

```bash
cd ~/Projects/EML/miso-data-archive
venv/bin/python run_export.py --no-wait
```

- At `Username for 'https://github.com':` type the GitHub username. With
  option B it already knows the username and asks only for the password.
- At `Password …:` paste the **token**, not your GitHub password, with Cmd+V.
  Nothing appears while you paste; that's normal. Press Return.
- If macOS asks whether `git-credential-osxkeychain` may use your keychain,
  enter your Mac password and click **Always Allow**. The unattended 8:30 job
  can't answer that question.

macOS saves the token in your login keychain, and the 8:30 job reuses it. If
an old, expired login is still saved, the first attempt fails with
*"Authentication failed"* and git discards it. Run the command again and git
asks for the new token. If there was nothing new to push, the run ends with
*"already up to date"* without asking; the token is then saved on the next
push that has data.

**4. Check on GitHub.** Open
<https://github.com/eml-lab/energy-dashboard-data>: the `miso/` folder should
contain `publish_date=…` folders.

**5. Turn on the daily 8:30 push** (once per machine):

```bash
bash scheduling/install_launchd_jobs.sh export
```

This loads `edu.lab.miso.export` and runs it once straight away. After a
minute, `launchctl list | grep edu.lab.miso` should show three lines with `0`
(the last exit code) in the middle column.

---

## The archive (unchanged)

The export only *reads* the archive and never modifies it. The archive jobs
work as before:

- `data/actual_load/interval_month=YYYY-MM/data.parquet`: native MISO rows
  `market_date, resolution, interval_start, interval_end, hour_ending,
  localResourceZone, load` (6 LRZ groups).
- `data/mtlf/publish_date=YYYY-MM-DD/data.parquet`: one immutable snapshot per
  **init**, with native rows `init, forecast_date, resolution, interval_start,
  interval_end, hour_ending, region, localResourceZone, loadForecast` (zones
  Z1–Z10, target days init+1 … init+7).

`README_OG.md` (in the project) covers setup, the API key, backfilling and the
storage guarantees.

### Checking completeness

```bash
venv/bin/python verify_archive_completeness.py   # writes archive_completeness_<date>.md
```

This is a read-only scan: it makes no API calls and never writes to `data/`.

- **What counts as complete:** each MTLF file must be **1680 × 9** (7 target
  days × 24 HE × 10 zones). Each actual-load file must be **(days in month ×
  24 × 6 LRZs) × 7**, with the current month counted through the latest
  published day.
- **What the report lists:** every file that deviates, with the exact missing
  target/operating days, hours (HE and EST clock time) and zones. It also
  reports duplicates, blank values and unexpected days, zones or columns.
- **Why gaps appear:** if MISO returns errors (e.g. HTTP 503) for some target
  days during a run, the MTLF job saves what it got, and the regular jobs
  never rewrite a saved file. Refill gaps with the existing tools, such as
  `fetch_one_mtlf.py`, which writes outside the archive.

## Project layout (`miso-data-archive`)

```
config.py                  # settings + .env loading (archive and export)
run_actual_load.py         # archive job: actual load        (3:00 AM)
run_mtlf.py                # archive job: MTLF               (8:00 AM)
run_export.py              # export job: push to GitHub      (8:30 AM)
verify_archive_completeness.py  # read-only completeness report for every archive file
src/
  miso_client.py           # MISO Data Exchange API client
  archive.py               # archive parquet storage (dedup, immutability)
  actual_load_job.py       # actual load passes
  mtlf_job.py              # MTLF passes
  dashboard_export.py      # archive -> cleaned daily export files (LRZ/MISO sums, day-ahead filter)
  git_publish.py           # commit + push from the managed clone
  export_job.py            # export orchestration (wait for data, build, push)
  cli.py, timeutils.py     # shared plumbing
scheduling/
  edu.lab.miso.{actualload,mtlf,export}.plist
  install_launchd_jobs.sh
  AUTOMATION.md
tests/
  test_archive.py          # archive immutability/dedup
  test_parsing.py          # API parsing/pagination
  test_export.py           # export tables, daily folders, git publishing (local repo, no network)
data/                      # the archive (gitignored)
exports/                   # local copy of the export: publish_date=D/ folders (gitignored)
dashboard_repo/            # managed clone of the shared repo (gitignored)
logs/                      # actual_load.log, mtlf.log, export.log, launchd_*.log
```

## Tests

```bash
venv/bin/python tests/test_export.py     # export tables, daily folders, git publishing
venv/bin/python tests/test_archive.py    # archive dedup + immutability
venv/bin/python tests/test_parsing.py    # API parsing, pagination, errors
```

None of these touch MISO or GitHub, and none need an API key.
