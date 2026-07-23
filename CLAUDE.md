# Motorhome Lifestyle

**Purpose:** Find the best integral/perfilada motorhome in the Canary Islands for a
family of four with two toddlers. Every day at 07:00 the pipeline searches, does deep
research, and refreshes the current week's **Top 5** on the dashboard. The board itself
is still WEEK-keyed (see `board.py`) — daily runs keep that week's Top 5 fresh with
whatever's new, they don't create a new section per day.

## Where things live

This IS the canonical, deployable location. As of 2026-07-20 the project was
consolidated here from a two-location split (`~/Developer/Manual Search Script Run/`
for code, `AI Coworking/.../Assets_HQ/Camper_Lifestyle/` for planning docs only) into
one place. There is no other copy — don't recreate the split.

- **GitHub repo:** `CommanderWi11/Motorhome_Search` (renamed 2026-07-20 from
  `Camper_Lifestyle`; git/API/web-UI access to the repo redirects automatically, but
  **the old Pages URL does NOT** — `commanderwi11.github.io/Camper_Lifestyle/` 404s
  immediately post-rename since the Pages subdomain is derived fresh from the current
  repo name. Update any bookmark to the new URL below.).
- **GitHub Pages serves `docs/` from `main`** (legacy mode, no Actions workflow).
  Pushing to `main` publishes the site: https://commanderwi11.github.io/Motorhome_Search/
- **launchd job:** `com.openbob.motorhome-search-daily` (renamed 2026-07-21 from
  `com.openbob.motorhome-search-weekly`, itself renamed 2026-07-20 from
  `com.openbob.camper-weekly`), symlinked into `~/Library/LaunchAgents/` from
  `launchd/com.openbob.motorhome-search-daily.plist` in this repo.
- **Logs:** `~/Library/Logs/motorhome-daily.log` (renamed from `motorhome-weekly.log`).

## The pipeline

    scripts/harvest.py        Stage A  scrape every source        -> scripts/candidates.json
    scripts/research-prompt.md Stage B  claude -p reads ads, web-searches, ranks
                                        -> scripts/winners.json
    scripts/apply_winners.py  Stage C  validate + fold into board -> docs/listings.json
    git push                  Stage D  Pages redeploys in ~60s

    scripts/weekly-search.sh            orchestrates all four
    launchd/com.openbob.motorhome-search-daily.plist   every day, 07:00 + 13:00/19:00 retries

**Run it now** (any time — it is idempotent per CALENDAR DAY, not per week):

```bash
launchctl kickstart -k gui/$(id -u)/com.openbob.motorhome-search-daily
tail -f ~/Library/Logs/motorhome-daily.log
```

To force a re-run on a day that already published, delete today's marker in `.state/`.

## Things that will bite you

- **GitHub Actions cannot run this.** The listing sites block datacenter IPs. The old
  `.github/workflows/weekly-search.yml` had a Monday cron and never once produced a
  listing. It is deleted. The job runs on the Mac, on purpose.
- **`claude -p` needs `--append-system-prompt`** with the headless override in
  `weekly-search.sh`. Without it the global CLAUDE.md startup protocol makes a headless
  run reply *"which workstation are we in today?"* instead of doing the work.
- **Never send `Accept-Encoding: br`** from the scrapers. `requests` cannot decode
  Brotli without the optional package, and you get binary garbage that every parser
  fails on *silently*.
- **`docs/listings.json` is a BOARD, not a feed.** Each entry is a past or present
  winner carrying the week it last won. Do not append to it — that is what
  `candidates.json` is for.
- **Supabase is currently dead** (project deleted; see `docs/supabase-setup.sql`). The
  dashboard falls back to localStorage and the weekly search reads
  `scripts/blocklist.json`, so nothing is broken — it just doesn't sync across devices.
  The Supabase table names (`camper_comments`, `camper_stars`, `camper_hidden`,
  `camper_status`) still carry the old `camper_` prefix — left as-is, since Supabase
  is dead anyway and renaming would require a live migration for no benefit.
- **A `claude -p` Stage B run can hang with zero progress and zero error.** Happened
  2026-07-20 (19:00 retry) and again 2026-07-23 (manual run) — 0% CPU, no open network
  connections, no session transcript file ever created under
  `~/.claude/projects/<encoded-repo-path>/*.jsonl`, for hours. Not a session-limit
  failure (which exits cleanly) — a true internal hang, root cause unconfirmed
  (candidates: iCloud placeholder eviction on a synchronous file read, an internal CLI
  deadlock — the repo's own scripts/CLAUDE.md files loaded fine on file inspection, so
  a plain missing-file explanation doesn't fully fit). **Fixed 2026-07-23**:
  `weekly-search.sh` now runs Stage B in the background under a plain-bash watchdog
  (`STAGE_B_TIMEOUT`, 25 min) and kills it on expiry — no `timeout`/`gtimeout` binary
  is installed on this Mac, hence the manual loop instead of a one-liner. A killed hang
  degrades to the normal "no winners.json" failure path, so the existing 13:00/19:00
  retry slots recover from it unattended instead of a human needing to notice and kill
  it. If this still doesn't fully explain future hangs, next step would be `fs_usage`
  or `dtruss` on the stuck PID to see exactly which syscall it's blocked in.
- **This folder is inside iCloud Drive** (`AI Coworking` is under `~/Library/Mobile
  Documents/com~apple~CloudDocs/`). iCloud can evict local files to cloud-only
  placeholders; this risk was raised and knowingly accepted during the 2026-07-20
  consolidation rather than pinning the folder "Keep Downloaded". If the unattended
  07:00 daily run ever fails specifically on missing/placeholder files, this is why.
- **Daily means up to 3x/day `claude -p` Stage B invocations, not up to 3x/week.**
  Switched from Monday-only to daily 2026-07-21 at Luis's request (see MEMORY.md) —
  worth knowing if Claude session limits start getting hit more often than the
  occasional Monday-morning one that happened before this change.

## The rubric

Family of four, two toddlers. Hard gates: **≥4 seatbelted travel seats with 3-point
belts** (the spec that silently disqualifies most cheap perfiladas — toddler car seats
need them), ≥4 berths, bathroom with shower, ≤3,500 kg (B licence), Canary Islands,
integral or perfilada. Then scored on family fit (40% — fixed beds and rear bunks are
everything; a bed you rebuild nightly around sleeping toddlers is a serious defect),
value (35%), condition/risk (15% — damp is the #1 killer of used motorhomes), and
Canary practicality (10% — ≤7 m for the roads and ferries).

Full rubric: `scripts/research-prompt.md`.

## Discarding

The 🗑 button on a card means **never show me this again, and never search for it
again** — `harvest.py` reads the discard list before it scrapes. From the terminal:

```bash
./scripts/discard.py <listing-id>       # discard
./scripts/discard.py --undo <listing-id>
./scripts/discard.py --list
```

## History

Full project history (force-push incident, integral/perfilada pivot, source
evaluations) lives in `MEMORY.md`. Original design docs from the 2026-05-11 build are
in `Resources/design-history/`.
