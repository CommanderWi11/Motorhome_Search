# Motorhome Lifestyle Memory

Last reviewed: 2026-07-21

## 2026-07-21 schedule change — Monday-only to daily

Luis asked to run the search daily instead of once a week. Clarified scope first since
the board is fundamentally WEEK-keyed (`board.py`: one position per ISO week) — two
readings existed: (a) refresh the current week's Top 5 daily, keeping week-grouping as
the UI model, or (b) switch the board itself to day-keyed sections. Luis chose (a).

- `board.py` / dashboard model: **unchanged**. Still one card per vehicle, positioned by
  the most recent ISO week it won. A same-week rerun just promotes into that week's
  existing entries — the cross-source dedup added earlier the same week (`same_vehicle`)
  applies here too, so daily reruns don't create duplicates either.
- `scripts/weekly-search.sh`: idempotency marker changed from `.state/<week>.done` to
  `.state/<date>.done` — was blocking every run after the first success of the week
  (by design, before this change); now blocks only repeats within the same calendar day.
  Commit message now includes the date (`chore: top 5 refresh <date> (week <week>)`) so
  daily republishes are distinguishable in git log.
- launchd: `com.openbob.motorhome-search-weekly` renamed to
  `com.openbob.motorhome-search-daily` (plist file, Label, log file all renamed to
  match); `StartCalendarInterval` dropped its `Weekday` key so all three slots
  (07:00/13:00/19:00 — the same retry pattern proven necessary the day before, see the
  hang incident below) fire every day instead of only Monday.
- **Cost/usage note:** this is up to 3x/day `claude -p` Stage B invocations now, not up
  to 3x/week — a real multiplier on Claude session usage. Worth watching if session
  limits start getting hit more often; the market itself (35-45 units total) barely
  turns over daily, so most days will republish the same winners and cost a research
  pass without changing what the family sees.

## 2026-07-20 consolidation — Motorhome_HQ/Motorhome_Search

The project had split into two diverged locations:
- `~/Developer/Manual Search Script Run/` — the actual canonical, deployable code.
- `AI Coworking/01_Personal_HQ/Projects/Assets_HQ/Camper_Lifestyle/` — an independent
  clone of the same GitHub remote, frozen on a 3-commit history from the 2026-05-19
  force-push incident (see below). Its own `CLAUDE.md` had been rewritten in place to
  flag itself as a dead clone, but that edit was never committed.

Consolidated into one canonical location, matching the existing `<Category_HQ>/<Project>/`
convention already used by `Assets_HQ/BSA_Options`:
- New path: `01_Personal_HQ/Projects/Motorhome_HQ/Motorhome_Search/` (this repo).
- GitHub repo renamed `Camper_Lifestyle` → `Motorhome_Search` (`gh repo rename`).
  Pages URL is now https://commanderwi11.github.io/Motorhome_Search/
- launchd job renamed `com.openbob.camper-weekly` → `com.openbob.motorhome-search-weekly`;
  log renamed `camper-weekly.log` → `motorhome-weekly.log`.
- Display/product name in the dashboard UI and docs is "Motorhome Lifestyle"; the
  folder/repo machine name is "Motorhome_Search" — mirrors the old
  folder-`Camper_Lifestyle`-displays-as-"Camper Life-style" pattern.
- Supabase table names (`camper_*`) were left unchanged — Supabase is dead anyway
  (see below), renaming would need a live migration for zero benefit.
- Original design docs (2026-05-11 plan + spec) preserved under
  `Resources/design-history/` — they only existed in the stale AI Coworking clone.
- Old clone archived at `Assets_HQ/Camper_Lifestyle_ARCHIVED_2026-07-20/`, then
  permanently deleted the same night once Luis confirmed nothing more was needed
  from it (everything of value was already merged in above).
- **iCloud risk accepted knowingly:** this repo now lives inside iCloud Drive
  (AI Coworking is under `~/Library/Mobile Documents/com~apple~CloudDocs/`). iCloud can
  evict local files to cloud-only placeholders between runs, which could break the
  unattended 07:00 Monday launchd run. Raised explicitly with Luis during planning;
  he chose to accept the risk rather than pin the folder "Keep Downloaded".

### Same-day incident: 19:00 retry hung, not failed
The 07:00 run on 2026-07-20 hit a Claude session limit (clean failure, no marker
written, correctly queued for retry). The 19:00 retry (last scheduled slot of the day)
instead **hung**: 2.5 hours elapsed, 0.19s total CPU time, zero open network
connections, and — the tell — **no Claude session transcript file was ever created**
under `~/.claude/projects/<encoded-path>/*.jsonl` for the whole run. A real `claude -p`
run writes to its session file continuously; total silence there (not just low CPU) is
the reliable signal of a true hang vs. a slow-but-working run. Killed the process tree
(`kill -TERM` on the `weekly-search.sh` PID and the `claude -p` child) and reran
`weekly-search.sh` manually — the manual run completed cleanly (`2026-W30.done`
written, board published, commit `63caf57` pushed) in well under the time the hung run
had already burned.

### Same-day fix: cross-source duplicate cards (commit a97152d)
Luis spotted it directly: the same physical **Etrusco 7400SB** was on the board twice
(`coches_net-0caa3cc4` week W29, `milanuncios-dcc75ac8` week W30) — same van, two ids,
because ids are `md5(source+url)` and the same vehicle gets a fresh id on every site it's
relisted on. The existing `fingerprint()` (exact token-set match) didn't catch it: one
title tokenized "7400 SB" as two tokens, the other tokenized "7400SB" as one, and the
descriptive words never overlapped at all ("garaje grande" vs "camas gemelas fijas").
Root-caused, fixed, and merged the live duplicate the same night:
- `_slug_tokens` now splits letter/digit runs so model codes match regardless of
  spacing ("7400SB" -> "7400"+"sb", same as "7400 SB").
- `_FP_STOPWORDS` expanded to drop base-chassis brands (fiat, ducato, ford, transit...)
  and engine/sale-status words (jtd, td, reservada...) — these were the dominant
  false-positive source (nearly every integral/perfilada shares a Fiat Ducato chassis,
  so two *different* models were sharing 4-6 "identity" tokens before this).
- New `harvest.same_vehicle(a, b)`: cross-source only (a dealer's own catalog can
  legitimately carry two units of one model — same-source near-dupes are never merged)
  plus a real token-overlap threshold, calibrated against the actual live dataset
  (candidates.json + board) to confirm zero false positives before shipping.
- `board.update_board()` now promotes into the existing card via `same_vehicle`, keeping
  the original id (so Supabase stars/comments/discards stay attached correctly).
- `apply_winners.py` now refuses to publish a week whose winners contain the same
  vehicle twice from different sources (a research-pass bug, not two real vans).
- Also: the dashboard's "✨ Nuevo" ribbon was purely session/visit-based (only showed
  on listings added since your last browser visit) — now `added_at` within the current
  top week always marks a card new, regardless of visit history or cleared localStorage.
- 9 new tests in `tests/test_harvest.py`, `test_board.py`, `test_apply_winners.py`
  pin down the real Etrusco case as a true positive and same-source/shared-chassis/
  conflicting-year cases as false-positive guards.

## 2026-05-24 recall improvements
Result: total dataset now ~5-6 motorhomes/run (vs. ~3 pre-pivot). Smaller gain than expected because Canary Islands market is genuinely thin.
- **Coches.net added** — heavy bot-detection, needs a humanlike browser context (locale/timezone/viewport + `navigator.webdriver` override). Pagination unreliable; single page only. Yields ~3-6 cards/run.
- **Autoscout24 dropped** — confirmed zero Spanish motorhomes on AS24.
- **Autocasion tried, dropped** — location filters are broken; returns nationwide listings regardless of param. No working Canarias filter exists.
- **Wallapop kept strict** — non-strict mode pulled 120 listings of which 95% were cars, real estate, and vans. Strict + brand whitelist is the right balance.
- **Brand whitelist** covers premium integral/perfilada manufacturers (hymer, bürstner, carthago, concorde, frankia, niesmann, morelo, benimar, chausson, adria matrix, dethleffs, etc.).
- **Key learning:** Wallapop is a keyword-fuzzy-match platform — non-strict mode is unworkable. Coches.net has aggressive bot detection. Autocasion's location filter is broken. Canary Islands motorhome supply is genuinely small (~10-15 active listings across all platforms at any time, later refined to 35-45 units across the whole archipelago once more sources were added — see harvest.py sources).

## 2026-05-24 pivot — integrales y perfiladas
- Family preference shifted away from camper-vans to **integrales** (Class-A coachbuilt) and **perfiladas** (low-profile). Vans, capuchinas, alcoba dropped.
- Price ceiling raised €55,000 → **€100,000** to capture this segment.
- Weight cap **kept at 3,500 kg** (B-licence constraint).
- Reference model updated from Sunlight Cliff (van) to Concorde Liner (gold-standard European integral).

## 2026-05-20 incident (the force-push that caused the later split)
- GitHub Pages was off (repo had been switched to private; free plan blocks Pages on private repos).
- Repo made public again; Pages re-enabled in legacy mode serving `main:/docs`.
- Remote `main` had been force-rewritten on 2026-05-15 to a fresh 3-commit history that wiped the May 12 triage/favorites/Autoscout24/hide-listings work. Local (`~/Developer/Manual Search Script Run/`) was canonical → force-pushed to restore.
- This is why the AI Coworking clone existed as a separate, never-updated copy of the pre-force-push-restore state for two months, until the 2026-07-20 consolidation above.

## Status
- [x] Project scaffolding
- [x] Weekly pipeline (harvest → claude -p research → validate → publish)
- [x] Dashboard (HTML + CSS + app.js), board model (not a feed)
- [x] GitHub Pages live at new URL
- [x] launchd automation, retry windows, idempotency
- [x] Single canonical location (2026-07-20 consolidation)
- [ ] Supabase re-provisioned (currently dead; dashboard runs on localStorage fallback)
- [ ] Wallapop selectors fixed (currently returns 0 candidates — DOM changed)

## Live URLs
- **Dashboard:** https://commanderwi11.github.io/Motorhome_Search/
- **GitHub repo:** https://github.com/CommanderWi11/Motorhome_Search

## Key decisions
- Single canonical location: `01_Personal_HQ/Projects/Motorhome_HQ/Motorhome_Search/` — code, docs, and dashboard together, no separate "planning only" copy elsewhere.
- GitHub Pages serves `docs/` folder from `main` branch (legacy mode, no Actions).
- Supabase: Family_Plan project, tables prefixed `camper_*` — dead since project deletion; not renamed.
- Wallapop API returns 403 unconditionally — use playwright headless Chromium instead; currently 0 candidates, selectors need fixing.
- `docs/listings.json` is a board (each id holds one position, its most recent winning week), not an accumulating feed.

## Sources (see also README.md and Resources/design-history/ for original design intent)
- **JSON APIs** (solid): Autocaravanas DM, Mundo Autocaravanas.
- **Playwright** (anti-bot): Milanuncios, Coches.net, Wallapop (currently 0 — broken selectors).
- **Static HTML**: Campermax, caravanas.net.
- **Via `claude -p` + WebFetch** (hostile markup): RentCamper Canarias (best family source — literas/bunk listings), Autocaravanas Canarias.
- ~~Autoscout24~~, ~~Autocasion~~ — dropped, see above.

## Open loops
- Wallapop harvester returns 0 candidates — DOM/selectors changed, needs investigation.
- Supabase project is dead (NXDOMAIN) — dashboard comments/stars/hidden state only persist to localStorage until re-provisioned via `docs/supabase-setup.sql`.
- GitHub Actions cannot run this pipeline (datacenter IP block) — always runs via local launchd.
