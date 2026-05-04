# WORKLOG

Append-only handoff notes for collaboration with other LLMs (Codex, future Claude sessions). Read at the start of substantive work; append a brief dated entry before ending substantive work. Diff shows _what_; this captures _why_.

Pre-v2.0.0 entries were pruned in the v2.0.0 release; institutional knowledge from those entries was migrated into `docs/internals.md` (deep-dive: architecture, dispatch model, failure modes, design history, recipes, anti-patterns). Per-version release detail lives in `CHANGELOG.md` (preserved verbatim).

---

## 2026-05-04 — v2.0.0 — superpowers-masterplan rebrand + Slice α intra-plan parallelism + Codex defaults on

**Scope:** Single coherent v2.0.0 release bundling six threads:

1. **Rename:** `claude-superflow` → `superpowers-masterplan`; `/superflow` → `/masterplan`; `commands/superflow.md` → `commands/masterplan.md`; `skills/superflow-detect/` → `skills/masterplan-detect/`; `hooks/superflow-telemetry.sh` → `hooks/masterplan-telemetry.sh`; `.superflow.yaml` → `.masterplan.yaml`. Hard-cut, no backward-compat aliases (per user instruction).
2. **Slice α of intra-plan task parallelism:** read-only parallel waves only — verification, inference, lint, type-check, doc-generation. Wave assembly + parallel SDD dispatch via Agent + wave-completion barrier in Step C step 2; single-writer status funnel + wave-aware activity log rotation in Step C 4d; files-filter union in Step C 4c; per-member outcome reconciliation (completed | blocked | protocol_violation) in Step C step 3; wave-count threshold in Step C step 5. Implementation tasks remain serial; Slice β/γ deferred per `docs/design/intra-plan-parallelism.md`.
3. **Codex defaults flipped to on:** `codex.routing: auto` (already default) + `codex.review: on` (was `off`). Step 0 codex-availability detection auto-degrades both to `off` for the run with one-line warning when codex plugin not installed. Doctor check #18 surfaces persistent misconfiguration as a Warning during lint.
4. **Codex documentation:** new top-level `## Codex integration` section in README (~490 words: why/how/defaults/install/disable/cross-references). Plugin.json description tweaked.
5. **Internal documentation:** `CLAUDE.md` (always-loaded ~620 words) + `docs/internals.md` (~8000 words, 15 sections: architecture, dispatch model, status format, CD rules, operational rules, wave dispatch, Codex integration, telemetry, doctor, verb routing, design history, recipes, anti-patterns, cross-references). Migrates institutional knowledge from pre-v1.1.0 WORKLOG entries that were pruned.
6. **History pruning:** deleted 5 pre-v1.1.0 spec/plan files (small-fixes, subcommands); trimmed WORKLOG to drop all v0.x and v1.0.0 audit-pass entries; the v1.0.0 entry's framing lives in CHANGELOG.

**Key decisions (the why):**

- **One v2.0.0 release instead of v1.1.0 + v2.0.0 split.** User picked "Bundle everything as v2.0.0" — the rename is breaking, justifies major version bump, and bundling avoids release-train churn. Just-committed v1.1.0 plan content (intra-plan-parallelism Slice α) was renamed in-place and folded into v2.0.0.
- **Hard-cut rename, no backward-compat aliases.** User instruction "Remove backwards compatibility alias" applied uniformly: no `/superflow` alias to `/masterplan`; no `.superflow.yaml` dual-load fallback (Step 0 only loads `.masterplan.yaml` / `~/.masterplan.yaml`). CHANGELOG migration notes call this out as a required user step. Reasoning: dual-load adds permanent maintenance burden + risk of silent stale-config-file confusion. Saved as durable feedback memory at `~/.claude/projects/.../memory/feedback_no_backward_compat_aliases.md`.
- **Slice α (read-only waves) over Slice β/γ.** Depth-pass on candidate mitigations found that the SDD-wrapper alone doesn't solve the central git-index-race for committing work — concurrent commits to the same branch race the index regardless of wrapper. Read-only work sidesteps it entirely. Slice β (~8-10d serialized commit) and Slice γ (~10-15d full per-task worktree subsystem) inherit the unsolved committing-work problem; deferred with sharpened, measurable revisit trigger in `docs/design/intra-plan-parallelism.md`.
- **Codex defaults flipped to on (auto routing + on review).** Most users who install Codex want adversarial review by default but had to explicitly enable it under v1.0.0. Graceful-degrade on missing-codex makes default-on safe (one-line warning, run continues with both off). New doctor check #18 catches persistent on-but-missing misconfiguration.
- **Internal docs written BEFORE history pruning** (Phase −1 of the v2.0.0 plan). The user's instruction to delete pre-v1.1.0 plans/worklogs would have lost the WORKLOG decisions/rationale captured across v0.2/v0.3/v0.4/v1.0.0 entries. Phase −1 migrated that knowledge into `docs/internals.md` §12 (Design decisions + deferred items) before Phase 0.1 deleted the source.
- **Aggressive WORKLOG trim.** User picked "trim WORKLOG to v1.1.0 entries only" (Q2 of the bundling AskUserQuestion). With v1.1.0 being rebadged as v2.0.0, this means only this entry remains. CHANGELOG retains the full release history.
- **CHANGELOG historical entries preserved verbatim** with an explanatory note at the top about the rename. Recommended Option (b) from the v2.0.0 plan's Risks section: simpler + more honest than rewriting historical attribution.
- **Bulk text rename via per-extension sed pass with explicit boundary regex** (`\bsuperflow\b`). Verification: zero surviving "superflow" references outside CHANGELOG.md after Task 0.3. Linter / IDE post-edit modifications (caught via system reminders) intentional and accepted.
- **Wave dispatch implementation order:** Phase 3 Tasks 1-12. Tasks 1 (cache extension) → 2-4 (wave assembly + 4-series + failure handling) — bundled into one commit since tightly coupled. Tasks 5-8 (doctor checks + B2 brief + flag + config) — bundled. Tasks 9 (hook), 10 (telemetry-signals), 11 (intra-plan-parallelism design doc), 12 (README) — separate commits. Task 14 (smoke verification) deferred — markdown-only project; documented as manual verification step in `docs/internals.md` §13 (Common dev recipes).
- **Hook portability sub-bug found and fixed during smoke test.** Original wave_groups extraction used gawk's `match()` with array argument (third arg) — gawk-only. Replaced with portable awk + grep + sed pipeline. Linux smoke-tested; first-turn caveat verified (tasks=0 when no prior record), wave detection verified (tasks=3 + waves=["verify-v2"] after delta).

**Operational notes:**

- 16 commits on the v2.0.0 release path:
  - Phase −1 (1): docs(internals) [97a74e4]
  - Phase 0 (4): housekeeping prune [c5798b5], git mv [3d84775], bulk text rename [3c25df2], plugin.json [7bcffdb]
  - Phase 1 (1): codex defaults + degrade + doctor #18 [c060f1f]
  - Phase 2 (1): README Codex section [9267b10]
  - Phase 3 (8): Tasks 1 [b720113], 2-4 [29ab357], 5-8 [36f8eb5], 9 hook [c660a85], 10 telemetry doc [c7cc929], 11 design doc [6d4fd60], 12 README [27e6d22]
  - Phase 4 (1, this entry + CHANGELOG): release v2.0.0
- Task list management via `TaskCreate` / `TaskUpdate`: 5 phase-level tasks for v2.0.0 (Phase −1 through Phase 4). Sub-tasks tracked in the plan file.
- Halt-mode discriminator suite re-grepped after each Step C / B-section edit — 25 references throughout, no orphans.
- Doctor table size: 18 rows (1-14 from v1.0.0 + 15-17 from Phase 3 Task 5 + 18 from Phase 1 Task 1.3). Step D parallelization brief says "all 18 checks" — matches.
- README structure: 16 top-level sections after v2.0.0 (added `## Codex integration` between `## Design philosophy` and `## Install`).
- Internal docs (`docs/internals.md`) committed BEFORE history pruning — institutional knowledge migrated successfully.
- `pwd` recovery needed once during a smoke test that `rm -rf`'d its tmpdir without `cd` first; no harm.

**Verification gaps (carried as v2.x followups):**

- **macOS hook smoke verification.** Linux smoke-tested only. Hook code is portable-by-construction (no GNU-only flags in the new wave_groups path; uses portable `head -n1` for find result, `stat -c '%Y' || stat -f '%m'` dual form for transcript-resolution fallback). README Option C softens the cross-platform claim.
- **Wave dispatch end-to-end smoke test.** Markdown-only project; the orchestrator IS the prompt. Dispatching a real wave requires a real `/masterplan execute` invocation against a hand-crafted parallel-group plan. Documented as manual verification step in `docs/internals.md` §13. Acceptance criterion #16 from the spec marked deferred until first user runs a real wave.
- **Codex graceful-degrade smoke.** Step 0 codex-availability detection logic landed as markdown logic (heuristic: scan system-reminder skills list for `codex:` prefix). Behavior in practice depends on Claude Code's actual context delivery. Worth verifying on first user run.
- **GitHub repo URL change is a manual user step.** plugin.json's `homepage` and `repository.url` updated to `https://github.com/rasatpetabit/superpowers-masterplan` (Phase 0.3's bulk rename + Phase 0.4's plugin.json metadata). User must rename the GH repo + update the plugin marketplace registration before the v2.0.0 install instructions work end-to-end. Flagged in plan's Risks; not blocking the local v2.0.0 commit/tag.

**Known followups (post-v2.0.0):**

- **Slice β/γ revisit trigger** — telemetry-derived; doctor check candidate (deferred to v2.0.x): scan recent plans for the trigger condition (≥3 parallel-grouped committing tasks, wall-clock >10 min). Telemetry fields `tasks_completed_this_turn` + `wave_groups` provide the data.
- **Codex CLI/API concurrency model verification** — affects whether a future slice could allow Codex tasks in waves (FM-4 is currently conservative).
- **Canned-`$ARGUMENTS` self-test specs** for routing-table drift detection. Markdown spec exercising every verb's branch.
- **Doctor check for the three-place verb-list invariant** (frontmatter description, reserved-verbs warning, routing table).
- **macOS smoke verification** of the renamed hook (gated on access to a macOS env).
- **Optional `/superflow` alias** — explicitly out of scope per user "no backward-compat aliases" rule. Users wanting old-name access install both plugins until they migrate.

**Branch state at end of v2.0.0:**

- 16 commits ahead of v1.0.0 on `main`.
- Tag `v2.0.0` created locally (Phase 4.4); push deferred to user-approval gate (Phase 4.5).
- Working tree clean.
- README Project status section updated to v2.0.0 framing.
