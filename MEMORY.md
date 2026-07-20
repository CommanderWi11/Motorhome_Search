# Motorhome Lifestyle Memory

Last reviewed: 2026-07-20

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
- Old clone archived (not deleted) at
  `AI Coworking/01_Personal_HQ/Projects/Assets_HQ/Camper_Lifestyle_ARCHIVED_2026-07-20/`.
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
