# PLAN.md — QuizLITE → native iOS app

**Audience: AI coding agents and the two humans (Fardeen, Aditya).** This document is the
single source of truth for what we're building, what was already decided and why, and what
order to do it in. If you are an agent starting a session: read this file and CLAUDE.md
first, then work the earliest phase with unchecked boxes — including Phase 0 desktop items —
skipping items tagged **[human]** (those need credentials, money, or a physical device).
When a phase mixes agent and human items, do every agent-doable item on the phase's branch
and list the pending human actions in the PR description. Do not relitigate the Decisions
section without being asked to.

Last updated: 2026-06-10. Evidence behind the decisions: [docs/AUDIT-2026-06.md](docs/AUDIT-2026-06.md).

---

## 1. Context and verdict (settled — do not re-derive)

QuizLITE v0 is the C++17/Qt6/SQLite desktop flashcard app in this repo, built summer 2024.
A June 2026 audit (22-agent review: code audit, market research, adversarial skeptics —
findings preserved in docs/AUDIT-2026-06.md) concluded:

- **The Qt desktop app is feature-frozen.** Desktop flashcard apps have no traction path in
  2026; the category's energy is mobile-first. Keep it building and green in CI; fix nothing
  except CI breakage. One sanctioned exception: the one-time Phase 0 port of the
  shortcuts-branch leak fix and menu bar; after Phase 0 closes, the freeze is absolute.
- **The C++ core is a spec, not a dependency.** It is ~2k LOC of string-concatenated SQLite
  CRUD with known bugs (catalogued in docs/AUDIT-2026-06.md). We **rewrite in Swift** and
  carry over the *behavior*, never the binary. Swift/C++ interop and Qt-for-iOS were both
  evaluated and rejected (interop tax is absurd for 2k LOC; Qt on iOS is non-native,
  LGPL-gray, and has zero successful precedent in this category).
- **Scheduling is FSRS, not our invention.** FSRS (the ML-derived scheduler Anki uses) is
  MIT-licensed with an official Swift port (`open-spaced-repetition/swift-fsrs`). Our old
  "lowest accuracy first" heuristic is retired. The per-question-stats idea survives two
  ways: the mastery heatmap is colored by FSRS-derived retention, and raw accuracy
  (`COUNT(grade != again) / COUNT(*)`) survives in the per-card history view and summary
  stats.
- **Distribution is the binding constraint.** Every winner in this market (Turbo AI,
  Coconote, Knowt, Gizmo) won on TikTok/SEO distribution, not engineering. The app only
  ships publicly if the content track (§6) is actually happening.

### Market facts the plan is built on (verified June 2026)

| Fact | Implication |
|---|---|
| Anki: free desktop, $24.99 iOS, brutal onboarding, zero gamification, FSRS built in | We match its scheduler for free (swift-fsrs) and beat it on first-session UX and price |
| Quizlet: paywalled Learn mode, ads, 1.4/5 Trustpilot | Trust posture (no ads, no account, no paywall-the-basics) is a real wedge |
| Knowt owns "free Quizlet alternative" (5M+ users) | Do NOT position as a free Quizlet clone |
| AI notes→flashcards apps are saturated; leaders openly "embrace churn" | Do NOT build AI card generation in v1; retention mechanics are the gap |
| Per-question analytics are paywalled at incumbents (Quizlet Plus, Brainscape Pro) | Our free per-card mastery heatmap is the flagship differentiator |
| Duolingo streak-anxiety backlash; 2025–26 hits were cozy/forgiving mechanics | If we gamify later, it's forgiving and mastery-gated, never punitive |
| Apple Guideline 4.3 punishes clone apps (17% of 2025 first submissions rejected) | Positioning + distinctive features matter for App Review, not just marketing |

## 2. Product definition

- **Working name:** QuizLITE (rename decision deferred to Phase 7; check App Store conflicts).
- **Target user:** high-school / college students who study on their phones, are annoyed by
  Quizlet's ads/paywalls, and bounce off Anki's UX.
- **One-sentence pitch:** *Offline flashcards with Anki-grade scheduling and the per-card
  stats everyone else paywalls — no account, no ads, no subscription.*
- **v1 differentiators (in priority order):**
  1. **Free per-card mastery heatmap** — visual per-card/per-deck mastery from real review
     history. Incumbents paywall this. Never paywall it — it's the wedge.
  2. **Trust posture** — offline-first, no account, no ads, no data leaves the device.
     One-time cosmetic "supporter unlock" later (Phase 8), never a subscription, never a
     cap on core study features.
  3. **FSRS scheduling** — parity with Anki's algorithm via swift-fsrs.
  4. **Low-friction first session** — from install to first review in under a minute
     (paste/CSV import, sane defaults, no settings wall).
- **Explicit non-goals for v1 (anti-scope — agents: do not build these):**
  - No accounts, no server, no sync (CloudKit sync is a v2 candidate, behind a flag).
  - No AI card generation (saturated; hallucination/trust problems; needs cloud).
  - No Android, no web, no iPad-optimized layout beyond what SwiftUI gives free.
  - No `.apkg` (Anki package) import — it's a multi-week subsystem; CSV + paste cover it.
  - No pet/creature, no streaks, no XP in v1. The "memory pet" concept was adversarially
    reviewed and refuted as the primary wedge (retention-fed pets starve when FSRS works
    well). Game *mechanics* may return post-v1 as forgiving, mastery-gated systems.
    Cosmetics that don't touch mechanics (alternate app icons, accent themes) are NOT
    gamification and are in scope (Phase 7).
  - No C++/Swift interop. No Qt anywhere near iOS.

## 3. Decisions (binding)

| Decision | Choice | Why |
|---|---|---|
| iOS code location | `ios/` directory in this repo | Plan, spec, and history in one place; agents work monorepo |
| UI | SwiftUI, iOS 17+ | Modern APIs, smallest surface for 2 part-time devs |
| Persistence | SQLite via **GRDB** | Closest to our existing schema thinking; testable; no CoreData/SwiftData migration lock-in |
| Scheduler | **swift-fsrs** (official open-spaced-repetition port) | MIT, FSRS-6, maintained; pin exact version in `Package.resolved` (committed) |
| Core data model | **Append-only per-review event log**, replay-from-log authoritative (see §5) | Lifetime counters (v0's TotalCorrect/TimesAsked) cannot feed FSRS, heatmaps, or future parameter optimization. The event log subsumes them |
| FSRS state | Derived by replaying the log through swift-fsrs (pure function); cached in `card_fsrs_state`, rebuildable | Stored derived state goes stale when FSRS params/algorithm change; replay never does. The cache exists because the due-queue query needs `due_at` per card |
| Primary study input | Self-graded flip cards: Again / Hard / Good / Easy | This is what FSRS consumes |
| Practice modes (MC / inverse MC) | Logged to review_log with their `mode`, but **excluded from FSRS state derivation in v1** — they feed history/stats only | FSRS has no weighted-review concept; inventing a grade mapping silently corrupts scheduling. Revisit post-beta with real data |
| Testing | Swift Testing framework + GitHub Actions macOS runner | Carry over v0's genuine strength: CI discipline |
| Monetization | Free during beta; one-time StoreKit "supporter unlock" = cosmetics only (alternate icons, accent themes). No deck caps, no feature gates | Subscription fatigue is documented; trust posture is the wedge |
| Old Qt app | Frozen (see §1 carve-out), kept building in CI | It's the working spec and the portfolio piece |

## 4. Phases

Work phases strictly in order; each has a Definition of Done (DoD). Check boxes as you
complete items (edit this file in the same PR as the work; until PLAN.md is merged to main,
stack work on the `chore/cleanup-june-2026` branch). If a phase reveals this plan is wrong
somewhere, update the plan in the same PR and say so in the PR description.

### Phase 0 — Repo hygiene (desktop side; agent work except where tagged)
- [x] Audit codebase; verify all README claims (docs/AUDIT-2026-06.md)
- [x] CLAUDE.md with build/test/lint commands
- [x] CMakeLists de-slopped (single GoogleTest path; SQLite3::SQLite3 + portability shim, commit e4ae498)
- [x] README rewritten to be accurate; LICENSE file added (MIT); PLAN.md + audit doc written
- [x] Local verification: 36/36 tests pass; clean build on macOS (Qt6, CMake 4.3)
- [ ] Commit all pending working-tree files (PLAN.md, LICENSE, docs/, README.md, CLAUDE.md) on `chore/cleanup-june-2026` **[agent — do this first if you find them uncommitted]**
- [ ] Push the branch **[human: push credentials]**, then confirm CI green on all 4 jobs — Ubuntu-Clang, Ubuntu-GCC, macOS, Linter; the Ubuntu jobs exercise the SQLite3 CMake shim from e4ae498 **[agent via `gh run list/view`]**
- [ ] CI hardening: pin `actions/checkout` to the current major (v4+) in all jobs; pin googletest to a release tag in the Ubuntu install step (currently clones master — time bomb)
- [ ] Port exactly two changes from `origin/shortcuts` onto the branch, by hand (NOT `git cherry-pick` — the branch is intermixed WIP): (1) the MainWindow leak fix, (2) the menu-bar setup. Both are described with file/line pointers in docs/AUDIT-2026-06.md §"Branch verdicts". Do NOT take `Syncing/*`, `Menu/Shortcuts.*`, or its CMakeLists/test changes
- [ ] Open PR to main; merge **[human]**; then delete `origin/shortcuts` and the merged cleanup branch **[human: remote deletion needs credentials]**
- **DoD:** main is green on all 4 CI jobs; PLAN.md/CLAUDE.md/LICENSE/audit doc live on main; no stale branches.

### Phase 1 — Spec extraction
- [ ] Create `ios/` directory; write `ios/SPEC.md`: precise behavior of the three study
      modes as implemented in `StudyingMethods/` (question selection: bottom-N by accuracy
      + random fill; option generation incl. the small-set `<4` padding behavior; inverse
      mode semantics), with **worked input→output examples for each mode and edge case**,
      written so Phase 5's property tests can be transcribed from them directly
- [ ] In SPEC.md: list the v0 bugs that must NOT be ported (use the named-bugs section of
      docs/AUDIT-2026-06.md: MC infinite loop, first-card skip, finish-button,
      shuffle-before-pad, apostrophe breakage, edit-destroys-history)
- [ ] In SPEC.md: map v0 schema → v1 schema (§5) with a worked example
- [ ] In SPEC.md: enumerate v1 screens (Library, Deck, Review, Practice, Heatmap, Import,
      Settings) with one paragraph each — no mockups needed, agents + SwiftUI iterate fast
- **DoD:** every study-mode behavior in SPEC.md has at least one concrete worked example;
  the bug list is complete against the audit doc; a reviewer can answer "what happens when
  a 3-card set runs MC mode?" from SPEC.md alone.

### Phase 2 — iOS scaffold + CI
- [ ] `ios/` Xcode project (SwiftUI app target + unit test target), iOS 17 deployment target
- [ ] Dependencies via SwiftPM: GRDB, swift-fsrs — commit `Package.resolved` (that file is
      the version pin of record; verify latest stable tags when integrating)
- [ ] GitHub Actions workflow: build + `xcodebuild test` on macOS runner for `ios/**` changes
- [ ] End-to-end spike test: create 10 cards, schedule with swift-fsrs, assert intervals
      are produced (proves the riskiest dependency on day one)
- **DoD:** CI runs iOS tests on every PR touching `ios/**`; the FSRS spike test passes.

### Phase 3 — Data layer (the foundation everything feeds on)
- [ ] GRDB schema + migrations per §5 (deck, card, review_log, card_fsrs_state)
- [ ] Store the database in an **App Group container** (decide the group identifier now) so
      the Phase 6 widget extension can read it without a migration
- [ ] Repository layer with full unit tests (CRUD, cascade deletes, apostrophes/quotes/
      emoji/CJK in card text — regression for v0's defining bug)
- [ ] FSRS replay: pure function `(reviewLog for card) -> (stability, difficulty, dueAt)`,
      golden-tested against swift-fsrs reference output; `card_fsrs_state` cache updated on
      every review write and rebuildable from scratch (test both paths agree)
- **DoD:** data-layer coverage ≥90% measured via `xcodebuild test -enableCodeCoverage YES`
  + xccov, asserted by a threshold script in CI; a seeded demo-DB fixture exists for UI work.

### Phase 4 — Review loop (the product core)
- [ ] Deck list + deck detail (create/rename/delete; card add/edit/delete; card edits never
      touch review history)
- [ ] Flip-card review screen: due queue from `card_fsrs_state.due_at`, Again/Hard/Good/Easy,
      each grade appends to review_log and refreshes the cache row
- [ ] Session end state + "what changed" summary (cards advanced, next due)
- [ ] Empty states and a 60-second first-run path (create deck → add 3 cards → review) with
      no settings required
- **DoD:** a full daily-study loop works on the simulator (agent-verified) and on a physical
  device **[human-verified]**; review_log is the only write path for study data.

### Phase 5 — Import + practice modes
- [ ] CSV import (header-optional, delimiter sniffing: comma/tab/semicolon)
- [ ] Paste-from-anywhere import: paste raw text, parse Q/A pairs with a preview-and-fix UI
      (covers "exported from Quizlet" — Quizlet's export is copy-to-clipboard text with
      configurable delimiters, not a file)
- [ ] Multiple choice + inverse multiple choice as **Practice** modes per SPEC.md (correct
      small-set behavior; property-test option generation against SPEC.md's examples — no
      infinite loops). Practice results append to review_log with `mode` set and are
      **excluded from FSRS derivation** (per §3); they appear in card history and stats
- [ ] Commit a real 100+ card Quizlet export as `ios/Tests/Fixtures/quizlet-export.txt`
      **[human: export it from a real Quizlet account once]**
- **DoD:** a 100+ card import — CSV file or pasted Quizlet export — completes in <10s with
  text fidelity (quotes, emoji, CJK), verified against the committed fixture.

### Phase 6 — Mastery heatmap (flagship differentiator)
- [ ] Per-deck grid heatmap: one cell per card, colored by FSRS-derived retention (from
      replay/cache, never from raw counters); tap → card history (which shows raw accuracy)
- [ ] Per-deck + all-decks summary stats (reviews, retention trend) — computed from
      review_log only
- [ ] Home-screen widget (WidgetKit): due-today count + mini heatmap, reading the App Group
      database (native hooks are an App Review differentiation signal, per Guideline 4.3 risk)
- **DoD:** heatmap renders from real review history; widget shows live data; everything in
  this phase is free forever.

### Phase 7 — Polish + beta
- [ ] App icon + the Phase 8 cosmetic set (2–3 alternate icons, accent themes) — built now
      so Phase 8 has something to sell
- [ ] Name decision: App Store search check for "QuizLITE" conflicts **[human decides]**
- [ ] Onboarding (≤3 screens, skippable); accessibility pass (Dynamic Type, VoiceOver on
      review screen)
- [ ] Apple Developer Program enrollment, TestFlight external beta setup, recruit 10+
      student testers **[human: account, $99/yr, recruitment]**
- **DoD:** beta build installable by non-developers; zero crashes in TestFlight/Xcode
  Organizer crash reports over the final two beta weeks across ≥10 active testers.

### Phase 8 — Ship
- [ ] StoreKit one-time "supporter unlock": the Phase 7 cosmetics only. Core study, import,
      and the heatmap stay free forever; no deck caps
- [ ] App Store listing written around the wedge (offline, no-account, free stats),
      screenshots from real decks; listing copy is agent-draftable, final approval **[human]**
- [ ] Privacy nutrition label: trivially clean (no data collected) — make it a marketing asset
- [ ] Submit **[human]**; expect Guideline 4.3 pushback risk; respond with differentiation
      evidence (heatmap, widget, offline posture)
- **DoD:** live on the App Store.

## 5. v1 data model (normative)

The single most important engineering decision. v0 stored lifetime counters; v1 stores
events and derives everything.

```
deck:            id, name, created_at
card:            id, deck_id (FK, cascade delete), question, answer, created_at, archived_at?
review_log:      id, card_id (FK, cascade delete), reviewed_at,
                 grade (again|hard|good|easy|practice_wrong|practice_right),
                 mode (flip|mc|inverse_mc), elapsed_ms?,
                 scheduled_interval_days?,          -- what the app told the user (event fact)
                 fsrs_stability?, fsrs_difficulty?  -- write-once TELEMETRY: what the
                                                    -- scheduler believed at review time.
                                                    -- No feature may read these.
card_fsrs_state: card_id (PK), stability, difficulty, due_at, fsrs_params_version
                 -- CACHE, rebuildable by replaying review_log through swift-fsrs.
                 -- The replay function is the source of truth; this table exists only
                 -- because the due-queue query needs due_at for every card.
```

Rules:
- `review_log` is append-only; edits to a card never touch its history (v0 destroyed
  history on edit — that was a bug, not a feature).
- FSRS state is **derived by replay** (pure function over a card's flip-mode rows). The
  per-row `fsrs_stability`/`fsrs_difficulty` columns are diagnostics, never inputs.
- Rows with `mode` in (`mc`, `inverse_mc`) are excluded from FSRS replay in v1 (§3); they
  feed card history and stats only.
- All reads that power scheduling, heatmaps, and stats derive from `review_log` (directly
  or via the sanctioned `card_fsrs_state` cache). No other denormalized counters.
- v0's `TotalCorrect`/`TimesAsked` per card maps to `COUNT(grade NOT IN (again,
  practice_wrong))` / `COUNT(*)` over the log — the old semantics are recoverable, the
  reverse is not.

## 6. Distribution track (runs parallel to Phases 2–8; humans only)

The audit's hardest finding: product quality does not get discovered in this category.
- Cadence: 3–5 short build-in-public videos/week (TikTok/Shorts/Reels) + occasional Reddit
  (r/GetStudying, r/Anki adjacents) starting **Phase 2**, not at launch.
- **Kill criterion (written here so it's honored):** if after 8 consecutive weeks of
  content there is no organic pull (followers, TestFlight signups, comments), stop public
  ambitions — finish the app as a portfolio piece and personal tool, and be glad.
- Expectation setting: indie ceiling in this category is roughly $0–5k/month. This is a
  side project dependent on agent labor; optimize for shipped-and-real over big.

## 7. How agents should work in this repo

- Read CLAUDE.md (commands/architecture) and this file at session start.
- Desktop app (`Database/`, `StudyingMethods/`, `Interface/`, `User/`): the remaining
  Phase 0 items ARE agent work; after Phase 0 closes, maintenance only — keep CI green,
  fix build breakage, never add features.
- iOS app (`ios/`): work the earliest incomplete phase; one phase per PR when feasible;
  check the boxes here in the same PR; keep `ios/SPEC.md` current when behavior decisions
  are made.
- Items tagged **[human]** need credentials, payment, a physical device, or a judgment call
  the humans reserved. Do everything else in the phase and enumerate the human items in the
  PR description.
- Every PR: tests for new logic, CI green, and an honest PR description (what's done,
  what's punted, what surprised you).
- If you find this plan contradicts reality (dependency dead, API changed, assumption
  falsified), update the plan in the same PR rather than silently working around it.
