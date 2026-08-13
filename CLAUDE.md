# Motorhome Lifestyle

**Purpose:** Find the best motorhome for a family of four (2 adults, a toddler and a
baby) — search scope is **all of Europe** (2026-08-11: restored back from the brief
2026-07-30 Canary-only detour, porting the geography forward rather than reverting,
so new+used search and every non-geography fix since stays intact; see MEMORY.md),
**new (0km/concesionario) or used, both searched extensively**. Every day at 03:00
the pipeline searches, does deep research, and publishes today's **Top 5** on the
dashboard. The board is Top 5 (today) + Favorites (starred) — no week-by-week
archive (`board.py` dropped the ISO-week model 2026-07-26): a listing that drops
out of the Top 5 and was never starred simply disappears on the next run.

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
    launchd/com.openbob.motorhome-search-daily.plist   every day, 03:00 (single run, no retries — 2026-07-30)

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
  degrades to the normal "no winners.json" failure path instead of a human needing to
  notice and kill it (2026-07-30: this used to fall through to the 13:00/19:00 retry
  slots — those are gone now, see below, so a killed hang today just means no fresh
  board until tomorrow's 03:00). **Also fixed 2026-07-23**: before killing, the watchdog now runs
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
  03:00 daily run ever fails specifically on missing/placeholder files, this is why.
- **Single 03:00 run, no same-day retries (2026-07-30).** Dropped the 07:00 +
  13:00/19:00 retry slots at Luis's request when refocusing search scope to the
  Canary Islands — back to one `claude -p` Stage B invocation/day, not up to 3x/day.
  Known risk: the 2026-07-27 incident below documents Claude's daily session limit
  resetting at 3:20am Atlantic/Canary — 03:00 is 20 min *before* that reset, so some
  days may still hit the limit. With no retry left, that means no fresh board that
  day; if it starts happening in the log, bump the plist to 03:25 (5 min after
  reset, the timing Luis originally chose for exactly this reason on 2026-07-27).

## The rubric

Family of four (2 adults, a 2.5-year-old, a 3-month-old). **Search scope is all of
Europe** (2026-08-11: restored from the 2026-07-30 Canary-only detour, ported
forward rather than reverted — geography goes back to Europe-wide, but every
non-geography lesson from both the 2026-07-26 rebuild and the Canary detour
stays: no body-type restriction, no invented percentage scoring, Top5+Favorites
board model, and new+used search). **Both new (0km/concesionario) and used are
searched extensively** — this started as a Canary-only-detour addition
(2026-07-30) and is kept now that scope is Europe-wide again; the original
2026-07-26 Europe-wide brief had been used-only. Hard gates: MAM ≤3,500 kg (B
licence), **length ≥ 6.90 m** (⚠️ do not revert to the old ≤7m preference, that
number was never re-requested), twin rear beds convertible to a double via a
factory infill kit, **left-hand drive**, ≥4 forward-facing 3-point-belt travel
seats. A 4th/5th child berth remains a strong preference, not a hard gate.
Bathroom is neither: the separate-bath/shower preference was removed at Luis's
request 2026-08-13 (`specs.bathroom_type` is still recorded as data, just never
scored). Logistics note: pan-European self-drive/ferry
framing is back — buy anywhere in Europe, self-drive it back, ferry only the
Canary leg (no distance/shipping-cost penalty by country) — and IVA/IGIC import
notes replace the local-IGIC-only note from the Canary detour.

**No body-type restriction** — carried over from the 2026-07-26 rebuild, still
correct: don't exclude capuchinas/campervans or require integral/perfilada.
(2026-08-12: added a soft tiebreaker on top — Stage B now favors integral over perfilada
when candidates are otherwise comparable, but this is a preference, not a filter; a
standout perfilada deal wins exactly as before. See `research-prompt.md`.) **No invented percentage scoring** — same, "rank by overall value" with no
weights/formula. **2 harvested sources, nationwide Spain again** (2026-08-11) —
`harvest.py`'s `SOURCES` is still just Milanuncios + Coches.net, and both URLs
are back to their nationwide-Spain form (no `/canarias.htm` or `/canarias/`
suffix — re-verified live via curl before landing, since they'd sat unused for
12 days). Everything else — every other European country, Autocasion, AutoScout24,
and live search for new-vehicle dealers anywhere in Europe — is Stage B's job
(live WebSearch/WebFetch), same division of labor as before.

Full rubric: `scripts/research-prompt.md`.

**Portal list**: `Resources/europe-motorhome-selling-sites.md` (added 2026-07-28,
briefly superseded 2026-07-30 by a Canary-only file during that detour, restored
2026-08-11 — the Canary file is left in the repo, marked superseded at its top,
not deleted) is the master list of Europe-wide selling sources Stage B works
through, in the order the file lists them (priority list, then country sections,
then active new-vehicle-dealer search). `weekly-search.sh` copies it into the
Stage B scratch dir alongside `candidates.json`/`config.js` since Stage B runs
isolated from the repo. Add new sites there, not by editing the portal list
inline in `research-prompt.md`.

## History view (manual shortlists)

2026-07-27: added a second, SEPARATE data path alongside the automated Top 5 +
Favorites board. Luis periodically runs (or receives) a deep multi-portal
research pass covering sites the automated harvester can't reach (mobile.de,
AutoScout24, leboncoin, Marktplaats, Subito, individual French/Italian/German
dealer sites, etc.) and pastes the result as a dated Top-5 markdown table. That
gets transcribed by hand into `scripts/ingest_manual_shortlist.py` (its pasted
tables sometimes carry OCR/copy corruption in cells and URLs — cross-reference
other dates' mentions of the same listing before trusting a truncated one) and
run to (re)build `docs/history.json`: one snapshot per date, ids/photos
generated the same way the automated pipeline does (`harvest.make_id`/
`fetch_og_image`, so a listing found manually and later found by the automated
harvester resolve to the identical id). `docs/app.js` renders one section per
date below Top 5/Favoritos, using the exact same card/star/delete code —
starring or deleting a listing collapses across every date that mentions it.

**This never touches `docs/listings.json`, `board.py`, or the daily pipeline.**
It's purely additive: a per-date archive for manual research, existing
alongside (not replacing) the automated Top-5-today model that intentionally
has no archive of its own. Don't conflate the two if extending either later.

## Discarding

The 🗑 button on a card means **never show me this again, and never search for it
again** — `harvest.py` reads the discard list before it scrapes, and Stage B checks
it again live before finalizing `winners.json` (added 2026-07-28, after a discarded
listing Stage B independently rediscovered via live web search got FATAL-rejected by
Stage C's validator — Stage B previously had zero awareness of discards at all, since
`candidates.json` filtering only protects harvester-sourced candidates, not Stage B's
own live web search). From the terminal:

```bash
./scripts/discard.py <listing-id>       # discard
./scripts/discard.py --undo <listing-id>
./scripts/discard.py --list
```

## History

Full project history (force-push incident, integral/perfilada pivot, source
evaluations) lives in `MEMORY.md`. Original design docs from the 2026-05-11 build are
in `Resources/design-history/`.
