# Motorhome Lifestyle

**Purpose:** Find the best motorhome for a family of four (2 adults, a toddler and a
baby) — search scope is now **all of Europe** (buy anywhere, self-drive it back as a
road trip, ferry the last leg to the Canaries). Every day at 07:00 the pipeline
searches, does deep research, and publishes today's **Top 5** on the dashboard. The
board is Top 5 (today) + Favorites (starred) — no week-by-week archive (`board.py`
dropped the ISO-week model 2026-07-26): a listing that drops out of the Top 5 and was
never starred simply disappears on the next run.

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

    scripts/harvest.py        Stage A  Milanuncios+Coches.net      -> scripts/candidates.json
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
- **`docs/listings.json` is Top 5 + Favorites, not a feed and not a week-archive.**
  An entry is on the board if it won today's Top 5 (`rank` 1-5) OR is starred
  (`rank: null`, a Favorite). Everything else is dropped by `board.update_board()`.
  Do not append to it — that is what `candidates.json` is for.
- **Supabase** — re-provisioning was in progress as of 2026-07-26 (old project was
  deleted; see `docs/supabase-setup.sql` for the schema). `harvest.py`'s
  `_supabase_blocklist()`/`_supabase_starred()` read `camper_hidden`/`camper_stars`
  straight from `docs/config.js`'s credentials — once a live project's URL/anon key
  land there, both the discard-veto and Favorites-retention paths work with no further
  code changes. Until then the dashboard falls back to localStorage (doesn't sync
  across devices) and the daily search falls back to `scripts/blocklist.json` /
  `scripts/starred.json`. The dashboard no longer uses `camper_comments`/
  `camper_status` (dropped in the 2026-07-26 rebuild) — the tables still exist in the
  schema, just unused; left as-is rather than migrated out.
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
  it. **Also fixed 2026-07-23**: before killing, the watchdog now runs
  `sample "$CLAUDE_PID" 5 -file .state/hang-sample-<timestamp>.txt` to capture every
  thread's call stack while the process is still stuck — both prior hangs were killed
  before anyone looked at what they were blocked on, so root cause was never more than
  a guess. If it hangs again, read that file first — it should show the actual blocked
  syscall (e.g. a file read stuck on iCloud vs. something inside the CLI itself)
  instead of needing a manual `fs_usage`/`dtruss` session after the fact.
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

Family of four (2 adults, a 2.5-year-old, a 3-month-old). Search scope is all of
Europe now, not Canary-only — sourced from a separate brief
(`motorhome-search-brief_2.md`, originally a Claude.ai Project) that superseded the
old Canary-only rubric on 2026-07-26. Hard gates: MAM ≤3,500 kg (B licence),
**length ≥ 6.90 m** (⚠️ flipped from the old ≤7m preference — do not carry the old
number over), twin rear beds convertible to a double via a factory infill kit,
**left-hand drive**, ≥4 forward-facing 3-point-belt travel seats. Bathroom (separate
preferred) and a 4th/5th child berth are strong preferences now, not hard gates —
neither is a Canary-only location requirement anymore. Logistics note: the family
self-drives the pickup as a road trip, so distance/country isn't penalized — only the
Canary ferry crossing is a real added cost.

**No body-type restriction** — the brief never asked to exclude capuchinas/
campervans or require integral/perfilada; that was purely an old-rubric holdover.
**No invented percentage scoring** — the brief says "rank by overall value" with
no weights/formula, so `research-prompt.md` asks for holistic judgment, not a
40/35/15/10-style rubric. **Only 2 harvested sources** (2026-07-26, third fix) —
`harvest.py`'s `SOURCES` is just Milanuncios + Coches.net (nationwide Spain, not
Canarias-only), matching the brief's §5 portal list exactly. Wallapop,
Autocaravanas DM, Mundo Autocaravanas, Campermax, caravanas.net, RentCamper
Canarias, and Autocaravanas Canarias were all removed — none were ever named in
the brief, and Mundo Autocaravanas alone had been 43% of the candidate pool. If
you're extending this prompt or the harvester later, resist re-adding any of
these three — all three crept back in once already from muscle memory (see
MEMORY.md for the full pattern across all three corrections).

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
