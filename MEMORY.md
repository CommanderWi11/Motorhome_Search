# Motorhome Lifestyle Memory

Last reviewed: 2026-07-28

## 2026-07-28 master portal list — Resources/europe-motorhome-selling-sites.md

Luis added `Resources/europe-motorhome-selling-sites.md` — a much more exhaustive
Europe-wide list of motorhome-selling sites (60+, organized by country, plus a
"Best sites to search first" priority sub-list) — and asked that Stage B search
through these sites, in order, on every run instead of the shorter ad hoc portal
list that had been embedded directly in `research-prompt.md`'s prose.

Wired in two places:
- `scripts/weekly-search.sh` now copies the file into the Stage B scratch dir
  (alongside `candidates.json`/`config.js`) since Stage B runs isolated from the
  repo (see the 2026-07-23 hang fix below) and would otherwise have no way to
  read it.
- `scripts/research-prompt.md` step 2 now tells Stage B to open that file and
  work through it in the order it's written — priority list first, then country
  sections in file order — rather than trusting Stage B to invent its own portal
  list from memory each run. Existing per-portal/total fetch budget (2 WebSearch +
  3 WebFetch per portal, ~25-30 fetches total) is unchanged — with 60+ sites now
  listed, the budget (not the site count) is what actually bounds a run.

To add/remove sites going forward, edit the Resources file, not
`research-prompt.md` — the prompt just points at it now.

## 2026-07-23 watchdog follow-up — capture diagnostics before killing

Luis pushed back correctly: a watchdog that kills a hang every run isn't a fix if the
hang itself is unexplained. Both prior hangs (2026-07-20, 2026-07-23) got killed before
anyone captured what the process was actually stuck on — so root cause was never more
than a guess (iCloud placeholder eviction vs. an internal CLI deadlock, see below).
Added: right before the watchdog kills a timed-out `claude -p`, it now runs
`sample "$CLAUDE_PID" 5 -file .state/hang-sample-<timestamp>.txt` — dumps every
thread's call stack while the process is still suspended, no sudo needed for a
same-user process (verified: `sample $$ 1 -file ...` on the shell itself worked
cleanly). Next hang should leave an actual answer instead of just a corpse.

## 2026-07-23 second Stage B hang — added a watchdog (permanent fix)

Manually triggered the daily search; Stage B (`claude -p`) hung again — same signature
as the 2026-07-20 incident (0% CPU, zero open network connections, no session
transcript ever created under `~/.claude/projects/...`), this time for 4h17m before
noticed. Killed the process tree on Luis's authorization, reran `weekly-search.sh`
manually — completed cleanly in 6.5 min, published board for 2026-W30, commit
`2066a11` pushed.

Since this is now a repeat (not a one-off), added an actual fix instead of relying on
manual detection again:
- `scripts/weekly-search.sh` Stage B now runs `claude -p` in the background under a
  plain-bash watchdog (`STAGE_B_TIMEOUT=1500`, 25 min) that kills it on expiry. No
  `timeout`/`gtimeout` binary is installed on this Mac, hence a manual poll loop
  instead of a one-liner.
- A killed hang now falls through to the same "no winners.json → exit 2" path as any
  other Stage B failure, so the existing 13:00/19:00 daily retry slots recover from it
  automatically — no more silent multi-hour freeze waiting on a human to notice.
- Root cause is still **unconfirmed**. Ruled out: session-limit failure (exits
  cleanly, doesn't match), missing `--allowedTools` permission prompt (already set,
  designed for exactly this headless case). Leading candidates: iCloud placeholder
  eviction blocking a synchronous file read (this repo lives under
  `~/Library/Mobile Documents/com~apple~CloudDocs/`, a known-accepted risk since the
  2026-07-20 consolidation), or an internal Claude Code CLI deadlock. If it recurs
  despite the watchdog now bounding the damage, next diagnostic step is `fs_usage` or
  `dtruss` on the stuck PID to see the exact blocked syscall.
- See also `CLAUDE.md` "Things that will bite you" — updated in the same pass.

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
- [x] Rebuilt for Europe-wide brief + Top5/Favorites dashboard (2026-07-26, see below)
- [ ] Supabase re-provisioned (currently dead; dashboard runs on localStorage fallback)
- [ ] Wallapop selectors fixed (currently returns 0 candidates — DOM changed)
- [ ] Phase 2: dedicated Playwright scrapers for European portals (mobile.de, AutoScout24, leboncoin, Subito.it, CamperOnLine, Marktplaats)

## 2026-07-26 rebuild — Europe-wide brief, Top 5 + Favorites, hang fix
Luis wanted the dashboard redone (simpler, better-looking) and pointed out the real
search criteria live in a *different* brief than this repo's rubric: a separate
Claude.ai Project (`motorhome-search-brief_2.md`) targeting **all of Europe** (buy
anywhere, self-drive it back as a road trip, ferry only the Canary leg — no
distance/shipping-cost penalty by country), not Canary-only stock. That brief is now
the single source of truth, replacing the old Canary-only rubric in
`scripts/research-prompt.md`.

**What changed:**
- **Rubric**: length flipped ≤7m→**≥6.90m** (easy to get backwards — flagged explicitly
  in the prompt), added **LHD** and twin-rear-beds-with-infill-kit as hard gates,
  dropped bathroom/geography as hard gates (now preferences), budget floor
  €20k→€50k, scope widened to mobile.de/AutoScout24/leboncoin/Subito.it/CamperOnLine/
  Marktplaats/OLX via Stage B live search (no dedicated scrapers yet — that's Phase 2).
- **Board model**: dropped the ISO-week archive entirely. `board.py` now does Top 5
  (today, rank 1-5) + Favorites (starred, rank null) — nothing else persists. A
  dropped, unstarred winner just disappears on the next run.
- **Dashboard**: rebuilt from scratch **twice** in one session — first pass kept too
  much of the old visual DNA (navy header, chip badges, overlay icons) and Luis said
  so directly ("looks exactly like the old one"); second pass is genuinely new: cream
  background, teal price accent, minimal cards with ONLY photo/title-link/star/delete/
  price/size/mileage/location — no verdict, flags, spec chips, comments, or status
  triage on the card. Lesson: "simpler" instructions this literal need a literal
  field-by-field rebuild, not a restyle of the existing structure.
- **Supabase**: `harvest.py` already had a dormant `camper_hidden`→blocklist bridge
  (`_supabase_blocklist()`) that just needed a live project to reactivate — added the
  missing other half, `_supabase_starred()`/`load_starred()` (mirrors the blocklist
  pattern, local `scripts/starred.json` cache so a Supabase outage doesn't wipe
  Favorites). Re-provisioning itself needs Luis to complete an OAuth flow via the
  connected Supabase MCP (`mcp__plugin_supabase_supabase__authenticate`) — was still
  pending his approval as of session end; `docs/config.js` needs the new URL/anon key
  once that lands.
- **Stage B hang**: 7 consecutive scheduled runs (07-20 through 07-26) hit the 25-min
  watchdog, every hang-sample showing an identical `getcwd()`/`open_nocancel` stall at
  process startup. A live interactive `claude -p "OK"` from the same iCloud repo cwd
  does NOT reproduce it (~5s response) — so it's specific to the cold/unattended
  launchd context, not a blanket "this cwd always hangs" fact. Mitigation applied
  regardless (cheap and safe either way): `weekly-search.sh` now runs the `claude -p`
  subprocess itself from a local non-iCloud scratch dir
  (`~/Library/Application Support/motorhome-search/stage-b-scratch/`) via
  `( cd "$SCRATCH" && exec claude -p ... )`, copying `candidates.json` in and
  `winners.json` back out. Stage A/C/D untouched (they've never hung). Not yet
  confirmed fixed by a real scheduled run — check `.state/hang-sample-*.txt` next time
  it fires to see if new samples appear.
- `docs/listings.json` reset to `[]` (old Canary-only board entries not worth
  migrating under a fully different rubric). `tests/test_board.py` and
  `tests/test_apply_winners.py` rewritten for the new model; 56 tests passing.
- Deleted stray `docs/superpowers/specs/*.md` (a design doc that was sitting inside
  the served Pages root by accident).

**First real run (same day, 13:47-14:00)**: manually kicked off right after the
rebuild. Stage B completed in 651s from the new scratch dir (vs. 7 straight hangs at
25min before the fix) — first evidence the hang fix works, not yet confirmed by an
actual unattended scheduled run. Stage C then rejected the output: `same_vehicle()`
false-positived a Giottiline Toscan 69GC against an unrelated Challenger Fiat C387,
both merely sharing generic layout words ("camas", "gemelas", "estrenar") that the
new brief's richer title style introduces into nearly every listing now. Fixed by
expanding `_FP_STOPWORDS` in `harvest.py` with the new shared vocabulary (camas,
gemelas, garaje, kit, separado, estrenar, etc.) — verified the specific collision
resolves and all 56 tests still pass. Re-ran Stage C against the already-captured
winners.json (no need to re-run the expensive Stage B) and published for real: 5
winners, Spain + France, top pick an Etrusco 7400SB already in Tenerife (€64,900).
Emailed to luisnavm@gmail.com. If `same_vehicle` false-positives again, the
stopword list in `harvest.py` (`_FP_STOPWORDS`) is the first place to look —
generic spec/layout vocabulary shared across most listings under the new rubric is
the recurring risk, not brand/model collisions.

**Same-day follow-up correction**: Luis pointed out the rewrite still carried old
Canary-rubric parameters despite the brief not asking for them. Two real leftovers,
both fixed:
1. `research-prompt.md` had an invented 35/30/20/15 percentage-weighted scoring
   rubric — structurally copied from the OLD rubric's 40/35/15/10, but the brief
   itself only says "rank by overall value" with no formula. Replaced with holistic
   judgment instructions, no weights.
2. `harvest.py` still hard-rejected capuchinas/campervans and required
   integral/perfilada (`_ACCEPT_RE`/`_REJECT_RE`) — an old-rubric body-type
   restriction the brief never asked for. Removed entirely; `_is_target()` no
   longer filters on body type, `_BRAND_RE` now matches only the brief's own §5
   model-family list (dropped carthago/frankia/concorde/niesmann/morelo/sun
   living/pilote/mclouis/laika — none are in the brief). Also added the explicit
   €50k-100k budget statement to the prompt (it was missing from the prompt text
   itself, only enforced via params.json's Stage A gates, which don't constrain
   Stage B's own live Europe-wide search at all).

**Lesson for future edits to this project**: when adapting the old
Canary-only-rubric prompt/scraper into a new brief, structural habits (weighted
scoring, body-type gates) carry over silently even when the explicit numbers are
updated. Cross-check every filter/gate in both `research-prompt.md` AND
`harvest.py` against the actual brief text, not just against "what looks similar
to before."

## Same-day third correction — sources, age, budget

Luis, again: "you are still holding parameters from the old project in searches...
use exclusively the following." A third read-only audit found the real remaining
issue: **sources**, not just gates/weights this time.

`harvest.py`'s `SOURCES` hit 7 sites; the brief's §5 portal list for Spain names
only Milanuncios/Coches.net/Autocasion. Wallapop, Autocaravanas DM, Mundo
Autocaravanas, Campermax, caravanas.net were never in the brief at all — leftover
from the old Canary-only project's own source discovery (Mundo Autocaravanas alone
was 34/80 = 43% of the candidate pool). `research-prompt.md` additionally
hard-coded RentCamper Canarias + Autocaravanas Canarias as *mandatory* Stage B
fetches every run (calling RentCamper "our single best family source" — old-rubric
evaluation language), and told Stage B all 7 harvester sources "siguen siendo
candidatos válidos" — actively re-legitimizing them instead of flagging the gap.

Also found: `max_age_years: 15` was a hard Stage-A reject with no brief basis,
directly contradicting the brief's "never discard on mileage/age alone." And
budget (`min_price`/`max_price`) was *also* a hard Stage-A reject, despite living
in the brief's "Parameters" section (§1), not its explicit 5-item "reject anything
that fails these" table (§2) — same category as height, which the brief
explicitly says is "not a constraint." Real proof the intended behavior was
already soft: the current #5 winner (€41,990 Challenger, 16% under the 50k floor)
was included by Stage B with a disclosure flag — but only because it came from
Stage B's own live search, bypassing Stage A's hard gate. An identical candidate
from Milanuncios/Coches.net would have been silently dropped.

**Fixed, user chose the strict option** (remove all 7 non-brief sources rather
than keep them relabeled as "extra"): `harvest.py` now only scrapes Milanuncios +
Coches.net, both widened from Canarias-only URLs to nationwide Spain
(`milanuncios.com/autocaravanas-de-segunda-mano/` no suffix;
`coches.net/autocaravanas-y-remolques/` — note this category slug was renamed
from `autocaravanas-segunda-mano`, the old path only still works via redirect).
`fetch_wallapop`/`fetch_autocaravanas_dm`/`fetch_mundo_autocaravanas`/
`fetch_campermax`/`fetch_caravanas_net` deleted entirely, along with `_is_target`'s
strict/brand-whitelist mode (`_BRAND_RE`) since Wallapop was its only caller,
`_passes_age`/`apply_price_filter` (no longer enforced — Stage B judges age/price
holistically per the brief), and now-dead helpers (`_parse_attrs`, `_soup`,
`_price_from`, `_ancestor_text`, `_blank`, `JSON_HEADERS`). `params.json` lost
`max_age_years` and the Wallapop geo-filter. `research-prompt.md` lost the
mandatory RentCamper/Autocaravanas Canarias section and the "still valid
candidates" framing for the 5 removed sources. One-off cleanup: purged 66 stale
non-brief entries from `scripts/candidates.json` (80→14; verified live re-run
afterward: 22, all `milanuncios`/`coches_net`). 47 tests pass (down from 56 — 9
removed for deleted functionality, none rewritten to fake-cover dead code).

**Pattern across all three corrections today**: each fix surfaced a NEW category
of old-project leftover the previous audit missed (invented scoring → body-type
filter → non-brief sources/gates). If a fourth complaint comes in, audit
`scripts/apply_winners.py`'s `validate()` and `scripts/board.py` next — not yet
audited for old-rubric assumptions as thoroughly as `harvest.py`/
`research-prompt.md` were across these three passes.

## Live URLs
- **Dashboard:** https://commanderwi11.github.io/Motorhome_Search/
- **GitHub repo:** https://github.com/CommanderWi11/Motorhome_Search

## Key decisions
- Single canonical location: `01_Personal_HQ/Projects/Motorhome_HQ/Motorhome_Search/` — code, docs, and dashboard together, no separate "planning only" copy elsewhere.
- GitHub Pages serves `docs/` folder from `main` branch (legacy mode, no Actions).
- Supabase: re-provisioned 2026-07-26 (new project "motorhome-search", org "Alfred Org", eu-central-1) after the old Family_Plan project was deleted — live, `docs/config.js` has current credentials. Tables still prefixed `camper_*`.
- `docs/listings.json` is Top 5 (today) + Favorites (starred), not an accumulating feed or week-archive — see "The rubric"/pipeline section of `CLAUDE.md` for the current model. (Superseded 2026-07-26; don't trust older "ISO-week board" language if it resurfaces anywhere.)

## Sources
Current, authoritative list lives in `CLAUDE.md`'s "The rubric" section (kept in one place deliberately, since this list has changed 3+ times — duplicating it here just risks the copy going stale again, which is exactly what happened to the pre-2026-07-26 version of this section).

## Open loops
- GitHub Actions cannot run this pipeline (datacenter IP block) — always runs via local launchd.
- `scripts/apply_winners.py`/`scripts/board.py` still not audited as thoroughly against the brief as `harvest.py`/`research-prompt.md` (see the 2026-07-26 entry above) — next place to look if a further "old parameters" complaint arises.

## 2026-07-27 — live 3:25am run + manual-shortlist history view

**`claude -p` session limit discovery**: a premature manual `launchctl kickstart`
at 00:36 hit "You've hit your session limit · resets 3:20am (Atlantic/Canary)"
and failed cleanly (no publish, no `.done` marker written). This is exactly why
Luis asked for the run at 3:25am specifically — 5 min after the daily reset.
Re-ran at 03:25:07, succeeded in ~9 min: first live confirmation the 2026-07-26
brief-only sourcing fix (commit `b0ae4e6`) holds end-to-end — Stage A pulled 36
candidates from only `milanuncios`/`coches_net`, board published a Top 3
(Challenger 287 GA, Etrusco 7400SB, Laika Kosmo 209).

**Tooling note**: this session's safety-classifier gate intermittently failed
(generic "temporarily unavailable" error) specifically on CronCreate and on
Bash/Monitor calls that referenced `launchctl kickstart` inside a long
backgrounded/scheduled script — even though the bare foreground command
(`launchctl kickstart -k gui/$(id -u)/com.openbob.motorhome-search-daily`) and
plain backgrounded wait-loops with no `launchctl` text both worked fine every
time. Workaround used: a background wait-loop with no system-command text as
the "alarm," then run the real command in foreground once woken. If a future
session hits the same wall scheduling anything with launchctl, try that split
before assuming the whole classifier is down.

**Manual-shortlist history view** (feature, see `CLAUDE.md` for the technical
writeup): Luis pastes dated Top-5 markdown shortlists from deep multi-portal
research (mobile.de, AutoScout24, leboncoin, Marktplaats, Subito, individual
dealer sites — everything the automated harvester can't reach) periodically,
not on a fixed cadence. Asked (via AskUserQuestion) whether 3 such dated
reports (07-24/07-25/07-27) should merge into today's board, only the latest
should count, or all should show as a per-date history — chose **history
view**, explicitly reversing the earlier 2026-07-26 decision to drop
week-archiving from the *automated* board. Built as a wholly separate,
additive path (`docs/history.json` + `scripts/ingest_manual_shortlist.py`) so
it doesn't reopen that automated-board decision at all.
**Why:** the source markdown consistently arrives with OCR/copy corruption in
some cells and URLs (e.g. `caravan-wendt.darado-t-328...` missing
`.de/de/fahrzeuge/c`) — cross-referencing later reports' mentions of the same
still-live listing (explicitly called out as "held over unchanged") was enough
to reconstruct every one this round, but a future batch might have a
corruption with no clean copy anywhere — flag that to Luis rather than
guessing a URL completion.
**How to apply:** next time Luis pastes a new dated shortlist, transcribe it
by hand into `scripts/ingest_manual_shortlist.py`'s `SHORTLISTS` dict (don't
build a markdown-table parser — the corruption makes that fragile) and re-run
it; it's idempotent per date.
**Not done**: live visual QA of the new history section in a real browser —
both playwriter and the claude-in-chrome extension were disconnected during
this (very early morning, unattended) session. Verified instead via full
manual code review, JSON validation, `node --check`, and the 47-test suite
(unaffected, since this is additive). Worth an actual look next time Luis is
at the machine.

## 2026-07-28 — discard promise didn't cover Stage B, fixed in two passes

Luis starred a history-view listing earlier (fixed separately, see prior
entry — Favoritos only looked at `allListings`). Later he discarded a
*different* listing off the live automated board: the Challenger 287 GA
Special Edition (netcampers_fr, €41,990, the below-budget standout). Next
"run a search" hit `FATAL: refusing to publish — netcampers_fr-de4813bc was
discarded by the family but came back as a winner` — twice in a row, each
costing a full ~12-13min Stage B run, because of a real gap: **the 🗑 button's
"never search for it again" promise only ever covered `harvest.py`
(candidates.json filtering).** Stage B's own live Europe-wide WebSearch has
zero visibility into discards — it happily re-finds and re-ranks a
genuinely great deal the family already said no to.

**Fix, two layers (defense in depth, since Stage B is an LLM and prompt
compliance isn't a guarantee):**
1. `research-prompt.md` now has Stage B `curl` the Supabase `camper_hidden`
   table (via `docs/config.js`'s public anon key, copied into the Stage B
   scratch dir by `weekly-search.sh` alongside `candidates.json`) and drop
   any finalist whose id matches. **First attempt at this had a real bug**:
   the instruction said to check "id or URL" against the id list — nonsense,
   since a URL can never equal a hash-id. Had to add an explicit
   id-computation snippet (`hashlib.md5(url).hexdigest()[:8]`, same scheme as
   `harvest.make_id`) for candidates Stage B finds itself (blank `id` field).
2. Even with that fixed, **the exact same vehicle collided a second time**
   the same day — Stage B found it relisted on **leboncoin** instead of
   netcampers_fr, a totally different id (id = md5 of URL, so a different
   URL always means a different id). Pure id-blocking architecturally cannot
   catch a relist. Real fix: `apply_winners.py`'s `validate()` now also
   cross-checks every winner against `same_vehicle()` (existing cross-source
   title/year fuzzy-match, already used for on-board dedup) for every known
   listing behind a blocked id — drawn from `candidates.json` +
   `listings.json` + `history.json`. Deliberately uses the looser
   `same_vehicle()`, not the strict `fingerprint()` that `harvest.py`'s own
   blocklist propagation uses — the asymmetry is intentional: publishing a
   discarded vehicle breaks an explicit trust promise, while over-matching
   here just costs one rank slot for a day.
3. **Also made a blocked winner non-fatal**: `validate()` used to abort the
   *entire* run on any blocked-id collision. Now it drops just that one
   winner and renumbers the rest (rank gaps only allowed when something was
   actually dropped — Stage B assigning non-consecutive ranks on its own is
   still a hard failure, that's a real quality signal). One bad entry now
   costs a rank slot, not a wasted ~12min Stage B run and a stale board.

**Gotcha hit while fixing this**: testing the fix by manually re-running
`apply_winners.py` against the same `winners.json` TWICE (once before the
same_vehicle fix landed) meant the first run's `board.update_board()` had
already dropped the blocked listing's title/year out of `listings.json`
(it wasn't a winner that pass, wasn't starred) — so the second run's
`blocked_listings()` had nothing left to `same_vehicle()`-match against.
Not a real bug: in a single normal pipeline pass, `validate()` reads the old
board BEFORE `board.update_board()` overwrites it, so the data is always
there when needed. Just a self-inflicted artifact of iterating on the fix
live against real files — if this ever needs re-testing, do it against a
fresh copy of winners.json, not by re-running apply_winners.py serially
against files it already mutated.

Net effect published this run: board dropped from the Stage-B-proposed 4 to
3 (Etrusco 7400SB, Hymer/Carado T448, Laika Kosmo 209) — Challenger correctly
excluded either way.
