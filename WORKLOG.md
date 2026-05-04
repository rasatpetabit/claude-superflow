# WORKLOG

Append-only handoff notes for collaboration with other LLMs (Codex, future Claude sessions). Read at the start of substantive work; append a brief dated entry before ending substantive work. Diff shows _what_; this captures _why_.

Pre-v2.0.0 entries were pruned in the v2.0.0 release; institutional knowledge from those entries was migrated into `docs/internals.md` (deep-dive: architecture, dispatch model, failure modes, design history, recipes, anti-patterns). Per-version release detail lives in `CHANGELOG.md` (preserved verbatim).

---

## 2026-05-04 — v2.0.0 — Slice α intra-plan parallelism + Codex defaults on + internal docs

**Scope:** Single coherent v2.0.0 release bundling five threads:

1. **Slice α of intra-plan task parallelism:** read-only parallel waves only — verification, inference, lint, type-check, doc-generation. Wave assembly + parallel SDD dispatch via Agent + wave-completion barrier in Step C step 2; single-writer status funnel + wave-aware activity log rotation in Step C 4d; files-filter union in Step C 4c; per-member outcome reconciliation (completed | blocked | protocol_violation) in Step C step 3; wave-count threshold in Step C step 5. Implementation tasks remain serial; Slice β/γ deferred per `docs/design/intra-plan-parallelism.md`.
2. **Codex defaults flipped to on:** `codex.routing: auto` (already default) + `codex.review: on` (was `off`). Step 0 codex-availability detection auto-degrades both to `off` for the run with one-line warning when codex plugin not installed. Doctor check #18 surfaces persistent misconfiguration as a Warning during lint.
3. **Codex documentation:** new top-level `## Codex integration` section in README (~490 words: why/how/defaults/install/disable/cross-references). Plugin.json description tweaked.
4. **Internal documentation:** `CLAUDE.md` (always-loaded ~620 words) + `docs/internals.md` (~8000 words, 15 sections: architecture, dispatch model, status format, CD rules, operational rules, wave dispatch, Codex integration, telemetry, doctor, verb routing, design history, recipes, anti-patterns, cross-references). Migrates institutional knowledge from earlier WORKLOG entries that were pruned.
5. **History pruning:** deleted 5 older spec/plan files (small-fixes, subcommands); trimmed WORKLOG to drop earlier entries; per-version release detail lives in CHANGELOG.

**Key decisions (the why):**

- **Slice α (read-only waves) over Slice β/γ.** Depth-pass on candidate mitigations found that the SDD-wrapper alone doesn't solve the central git-index-race for committing work — concurrent commits to the same branch race the index regardless of wrapper. Read-only work sidesteps it entirely. Slice β (~8-10d serialized commit) and Slice γ (~10-15d full per-task worktree subsystem) inherit the unsolved committing-work problem; deferred with sharpened, measurable revisit trigger in `docs/design/intra-plan-parallelism.md`.
- **Codex defaults flipped to on (auto routing + on review).** Most users who install Codex want adversarial review by default but had to explicitly enable it under v1.0.0. Graceful-degrade on missing-codex makes default-on safe (one-line warning, run continues with both off). New doctor check #18 catches persistent on-but-missing misconfiguration.
- **Internal docs written BEFORE history pruning.** The user's instruction to delete earlier plans/worklogs would have lost the WORKLOG decisions/rationale captured across earlier entries. A migration step moved that knowledge into `docs/internals.md` §12 (Design decisions + deferred items) before the source was deleted.
- **Aggressive WORKLOG trim.** User picked to keep only the v2.0.0 entry. CHANGELOG retains the full release history.
- **Wave dispatch implementation order:** Tasks 1 (cache extension) → 2-4 (wave assembly + 4-series + failure handling) — bundled into one commit since tightly coupled. Tasks 5-8 (doctor checks + B2 brief + flag + config) — bundled. Tasks 9 (hook), 10 (telemetry-signals), 11 (intra-plan-parallelism design doc), 12 (README) — separate commits. Smoke verification deferred — markdown-only project; documented as manual verification step in `docs/internals.md` §13 (Common dev recipes).
- **Hook portability sub-bug found and fixed during smoke test.** Original wave_groups extraction used gawk's `match()` with array argument (third arg) — gawk-only. Replaced with portable awk + grep + sed pipeline. Linux smoke-tested; first-turn caveat verified (tasks=0 when no prior record), wave detection verified (tasks=3 + waves=["verify-v2"] after delta).

**Operational notes:**

- 16 commits on the v2.0.0 release path; tasks tracked via `TaskCreate` / `TaskUpdate` (5 phase-level tasks).
- Halt-mode discriminator suite re-grepped after each Step C / B-section edit — 25 references throughout, no orphans.
- Doctor table size: 18 rows. Step D parallelization brief says "all 18 checks" — matches.
- README structure: 16 top-level sections after v2.0.0 (added `## Codex integration` between `## Design philosophy` and `## Install`).
- Internal docs (`docs/internals.md`) committed BEFORE history pruning — institutional knowledge migrated successfully.
- `pwd` recovery needed once during a smoke test that `rm -rf`'d its tmpdir without `cd` first; no harm.

**Verification gaps (carried as v2.x followups):**

- **macOS hook smoke verification.** Linux smoke-tested only. Hook code is portable-by-construction (no GNU-only flags in the new wave_groups path; uses portable `head -n1` for find result, `stat -c '%Y' || stat -f '%m'` dual form for transcript-resolution fallback). README Option C softens the cross-platform claim.
- **Wave dispatch end-to-end smoke test.** Markdown-only project; the orchestrator IS the prompt. Dispatching a real wave requires a real `/masterplan execute` invocation against a hand-crafted parallel-group plan. Documented as manual verification step in `docs/internals.md` §13. Acceptance criterion #16 from the spec marked deferred until first user runs a real wave.
- **Codex graceful-degrade smoke.** Step 0 codex-availability detection logic landed as markdown logic (heuristic: scan system-reminder skills list for `codex:` prefix). Behavior in practice depends on Claude Code's actual context delivery. Worth verifying on first user run.

**Known followups (post-v2.0.0):**

- **Slice β/γ revisit trigger** — telemetry-derived; doctor check candidate (deferred to v2.0.x): scan recent plans for the trigger condition (≥3 parallel-grouped committing tasks, wall-clock >10 min). Telemetry fields `tasks_completed_this_turn` + `wave_groups` provide the data.
- **Codex CLI/API concurrency model verification** — affects whether a future slice could allow Codex tasks in waves (FM-4 is currently conservative).
- **Canned-`$ARGUMENTS` self-test specs** for routing-table drift detection. Markdown spec exercising every verb's branch.
- **Doctor check for the three-place verb-list invariant** (frontmatter description, reserved-verbs warning, routing table).
- **macOS smoke verification** of the telemetry hook (gated on access to a macOS env).

**Branch state at end of v2.0.0:**

- 16 commits ahead of v1.0.0 on `main`.
- Tag `v2.0.0` created locally (Phase 4.4); push deferred to user-approval gate (Phase 4.5).
- Working tree clean.
- README Project status section updated to v2.0.0 framing.

---

## 2026-05-04 — v2.1.0 — README polish + gated→loose offer + Roadmap

**Scope:** Additive release on the v2.x track; no breaking changes. Three threads:

1. **README polish:** reordered `## Why this exists` to precede `## What you get`; appended a 6-bullet benefits paragraph to "Why this exists" (long-term complex planning, aggressive context discipline, dramatic token reduction, parallelism for faster operation, cross-session resume, cross-model review); added `### Defaults at a glance` sub-section under `## Configuration` (~50-line compact YAML block); added `## Roadmap` top-level section before `## Author` (6 deferred items + 4 documented non-features, each with measurable revisit trigger).
2. **Gated→loose switch offer:** new AskUserQuestion at Step C step 1 (after telemetry inline snapshot, before per-task autonomy loop). When `autonomy == gated` AND `config.gated_switch_offer_at_tasks > 0` AND plan task count ≥ threshold (default 15) AND not already dismissed, offer 4-option switch. Two new status frontmatter fields handle suppression: `gated_switch_offer_dismissed` (permanent per-plan) + `gated_switch_offer_shown` (per-session; re-fires on cross-session resume by design).
3. **Release bookkeeping:** plugin.json 2.0.0 → 2.1.0; CHANGELOG `[2.1.0]` block; this WORKLOG entry; tag + push.

**Key decisions (the why):**

- **Top-level `gated_switch_offer_at_tasks` config key, not nested under `autonomy:`.** Initial plan suggested `autonomy.gated_switch_offer_at_tasks` but YAML doesn't allow `autonomy: gated` (scalar) AND `autonomy: { gated_switch_offer_at_tasks: 15 }` (block) to coexist. Renaming the existing scalar would be a breaking change. Top-level key avoids the conflict. Cost: less namespaced, but unambiguous and additive.
- **Per-session `gated_switch_offer_shown` flag re-fires on cross-session resume by design.** Reasoning: when the user comes back to a paused plan after a break, they may have changed their mind about the gated friction. Asking once per session is the right cadence. Permanent dismissal is available via `gated_switch_offer_dismissed: true` for users who DON'T want re-prompting.
- **The orchestrator does NOT modify `.masterplan.yaml`** even when the user picks "Switch + don't ask again on any plan." It writes a `## Notes` entry recommending the change; user takes the action manually. Per CD-2 — config files are user-owned.
- **Default threshold = 15 tasks.** Educated guess. The audit-pass v1.0.0 plan was 12 tasks; the v1.1.0 wave-dispatch plan was 14 tasks; the v2.0.0 release plan was effectively 18+ tasks. 15 captures the "this is going to take a while" point. Easy to tune via config.
- **Reordered Why before What in README per user instruction.** Better reading flow: explain the value before the verb surface. The "thin orchestrator" framing in the tagline paragraph still introduces *what* it does at a high level, then "Why this exists" goes deep on value, then "What you get" enumerates the verbs.
- **Roadmap section frames deferred items as "decided NOT to ship yet, and why"** rather than "promised future work." Prevents the section from being read as a feature-request invitation. Each item has a measurable revisit trigger (per the existing `docs/design/intra-plan-parallelism.md` convention).
- **Defaults at a glance duplicates the Configuration section's schema.** Maintenance cost: when a default changes, both must update. Mitigation: keep the at-a-glance block deliberately compact (just `key: value` pairs, no inline explanation) so it's easy to eyeball-diff against the full schema.

**Operational notes:**

- 3 commits this release: Phase 1 README polish [9226037], Phase 2 gated→loose offer [9d22c5d], Phase 3 release (this commit). Smaller release than v2.0.0.
- Halt-mode discriminator suite: 26 unique lines (was 25 pre-Phase 2). The +1 is from the new Step C step 1 paragraph; not an orphan reference, just a contextually-correct mention.
- README structure verified post-changes: `## Why this exists` (line 9), `## What you get` (line 24), `## Configuration` with `### Defaults at a glance` + `### Full schema (with explanations)` sub-sections, `## Roadmap` (line 620) before `## Author`.
- Status file format docs updated in 3 places: README "Status file" section, commands/masterplan.md "Status file format" section, docs/internals.md §4 "Status file format". All three include both new optional fields with v2.1.0+ annotation.

**Verification gaps (carried as v2.x followups):**

- **Gated→loose offer behavior not yet first-user-tested.** Markdown logic; runtime behavior depends on a real `/masterplan execute` against a 15+ task plan under autonomy=gated. First-user verification is the smoke test. Documented in WORKLOG (this entry) + plan file's "Risks" section.
- Same gaps as v2.0.0 carried forward: macOS hook smoke verification, canned `$ARGUMENTS` self-test specs.

**Known followups (post-v2.1.0):**

- **Telemetry signal for "user dismissed the offer"** — would help inform whether threshold default 15 is well-calibrated. Add `gated_switch_offer_outcome: switched|stayed|never_offered` field to the Stop hook's record. Defer until a real signal exists from a few users.
- **Doctor check for unusually-long-but-still-gated plans** — flag plans where `task_count > 20` AND `autonomy: gated` AND `gated_switch_offer_dismissed: true` for >7 days as candidates for re-revisit. Niche.
- All v2.0.0 followups still apply (Slice β/γ trigger doctor check, Codex concurrency verification, $ARGUMENTS self-tests, macOS smoke).

**Branch state at end of v2.1.0:**

- 3 commits ahead of v2.0.0 on `main`.
- Tag `v2.1.0` created locally (Phase 3); push deferred to user-approval gate.
- Working tree clean.
- plugin.json: 2.1.0; description mentions the v2.1.0 surface.
