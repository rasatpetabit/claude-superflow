# WORKLOG

Append-only handoff notes for collaboration with other LLMs (Codex, future Claude sessions). Read at the start of substantive work; append a brief dated entry before ending substantive work. Diff shows _what_; this captures _why_.

---

## 2026-05-01 — `/superflow` v0.2.0 small-fixes pass (`feat/superflow-small-fixes`)

**Scope:** Bundled six findings from a `/superflow` analysis session into one improvement pass. Spec: `docs/superpowers/specs/2026-05-01-superflow-small-fixes-design.md`. Plan: `docs/superpowers/plans/2026-05-01-superflow-small-fixes.md` (8 tasks). Status: `docs/superpowers/plans/2026-05-01-superflow-small-fixes-status.md`. All 8 tasks complete; v0.2.0 released.

**Key decisions (the why):**

- **Bundled six small fixes into one pass instead of one per finding.** Each fix is independently small and several share the same files (`commands/superflow.md`, `README.md`, `CHANGELOG.md`). Six separate spec/plan/execute cycles would have been pure overhead. Six-in-one preserves a coherent "v0.2.0" release rather than a stream of patch versions. The advisor explicitly recommended this framing during analysis ("present the findings, don't brainstorm" — then user picked the bundled-spec option).
- **Three larger threads were deliberately deferred to dedicated specs:** SDD × Codex routing per-task loop boundary (analysis finding #3), 4-review pile-up under default + codex-review (#5), intra-plan task parallelism (#12). The boundary one is the highest-impact orchestration ambiguity; tackle it before broadening Codex use further.
- **`codex_routing: off` for THIS execution run** — the SDD × Codex boundary is unresolved at the time of this run, and this pass doesn't fix it. Setting `off` sidesteps the ambiguity. Future plans (after the boundary is resolved) can use `auto`. The plan's per-task `**Codex:** ok|no` annotations are valid documentation regardless — they'll take effect once the boundary is settled.
- **Inline execution instead of subagent dispatch** — plan tasks are mechanical text edits with explicit Edit + grep verification. Spinning up 8 subagents would re-load the same context per task. The "Subagents do the work" pillar applies to LONG runs where orchestrator context bloats; this pass fits comfortably in one session.
- **Behavior change made the default rather than opt-in.** The user's permissiveness ask drove this: default `gated` mode no longer prompts on pre-configured Codex automation. Users who want the old chatty behavior set `codex.confirm_auto_routing: true` and `codex.review_prompt_at: "low"`. Documented as a behavior change in CHANGELOG `[0.2.0]` Changed.
- **Step 4b's SHA fallback bug was real**, not theoretical. Verified empirically: `git merge-base HEAD master` returns the HEAD SHA when on master tip. Fix removes the fallback entirely; `task_start_sha` is now required in implementer return digest, blocks recoverably if missing.

**Operational lessons (worth keeping in mind):**

- Multiple Edits to the same file in one session work fine sequentially — the Edit tool's "must read before write" check holds within a session. But moving across worktrees (e.g., the .gitignore on main vs. files in the worktree) requires re-reading per worktree.
- The advisor (when applicable) is especially good at re-framing: it caught that the user wanted "the analysis as deliverable" rather than "let's brainstorm together," which would have wasted an hour of one-question-at-a-time refinement.
- Gated checkpoints between tasks in a small pass like this are noise — the user said "go" once and that was standing approval. Long autonomous runs (`/loop`) need different tradeoffs.
- `git status --porcelain` is correctly never cached in `git_state` (per CD-2). Confirmed in this pass: every Step C entry that reads dirty state goes live.

**Open questions / followups:**

- The SDD × Codex boundary (analysis finding #3) needs its own spec. Without it, `codex_routing: auto` under SDD has undefined semantics — superflow inlines tasks via SDD, but Codex routing is per-task and superflow-decided in Step C 3a, with no documented mechanism for the orchestrator to intercept before SDD dispatches.
- `superpowers:writing-plans` skill upstream doesn't know about `**Codex:** ok|no` annotations. We documented the convention in superflow's Step B2 brief — plans authored via `/superflow` will get annotations. Plans authored elsewhere won't. Worth proposing an upstream PR to writing-plans at some point.
- Telemetry per-task model usage (analysis finding #11) wasn't included in this pass. Small but isolated change; would inform tuning of `codex.max_files_for_auto` and the eligibility heuristic.
- `finishing-a-development-branch` is mandatorily interactive even under `--autonomy=full` (analysis finding #10). Not fixed in this pass.

**Branch state at end of pass:**

- 11 commits ahead of `main` on `feat/superflow-small-fixes`.
- Linear history, no merge commits, no rebase needed.
- `.worktrees/` ignored on main; .worktrees/superflow-small-fixes is the active worktree.
- Suggested next: invoke `superpowers:finishing-a-development-branch` to merge to main or open a PR.

---

## 2026-05-02 — `/superflow` v0.3.0 explicit phase verbs

**Scope:** Added `new`, `brainstorm`, `plan`, `execute` as explicit first-token verbs in `/superflow`. Spec: `docs/superpowers/specs/2026-05-02-superflow-subcommands-design.md`. Plan: `docs/superpowers/plans/2026-05-02-superflow-subcommands.md`.

**Key decisions (the why):**

- **Discoverability over phase-control framing.** User picked "Discoverability — make verbs visible at a glance" as the motivation. The phase-control verbs (`brainstorm`, `plan`) fall out for free once the verbs are addressable, but they aren't the headline.
- **Additive, no deprecation.** Bare-topic catch-all and `--resume=<path>` keep working forever. Existing `/loop /superflow <topic> ...` invocations and any cron / docs that use the bare-topic form continue unchanged. Cost: routing logic remains "verb match OR catch-all."
- **`halt_mode` as a tiny internal state machine instead of a per-step flag.** Set once in Step 0 from the verb match, read by B1/B2/B3/C. Cleaner than threading four boolean flags through every dispatch site, and the in-session "Continue to plan now / Start execution now" overrides become a simple `halt_mode` flip.
- **`plan --from-spec=<path>` skips Step B0 — spec's location is authoritative.** B0a covers the trunk-branch foot-gun (relocate spec to a feature worktree if it lives on main/master/trunk). Caught during spec self-review; without it, we'd silently inherit the trunk branch and only discover SDD's refusal at execute time.
- **`/superflow plan` (no args) does a Step P picker, not an error.** User flagged "list recent specs without a plan, let user pick" as the desired behavior. One filesystem scan beats forcing the user to remember/type the path; consistent with how `/superflow` (empty) lists in-progress plans.
- **Verb tokens reserved.** Topics literally named `new`, `brainstorm`, `plan`, `execute` need a leading word. Documented in the README. Concrete cost is small; alternatives (escape character, `--topic=` flag) would have introduced more grammar than they saved.

**Operational notes:**

- All edits are markdown-only. No code, no automated test suite for the prompt. Verification is grep-based per-task plus a final smoke-read of the modified `commands/superflow.md`.
- The README's existing `## Subcommand reference` got a new `### Verbs` subsection at its top; the original table is now `### Invocation forms (back-compat detail)`. README structure preserved otherwise.
- v0.3.0 is a minor version bump because the externally-visible grammar grew (new verbs) without breaking anything that already worked.

**Open questions / followups:**

- The `/loop /superflow brainstorm <topic>` foot-gun is mitigated by a one-line warning at Step 0; a stricter "auto-disable loop under halt_mode" could be considered later if telemetry shows users still hit it.
- `/superflow execute` with zero in-progress plans currently routes to Step A which offers "Start fresh" → kickoff. That's slightly indirect under explicit-verb framing; a future polish could reword the option to "No in-progress plans. Run /superflow new <topic>?" so the verb model stays coherent.
