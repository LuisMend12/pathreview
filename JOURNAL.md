# JOURNAL

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/47

**Issue title:** Agent state isn't persisted across API restarts, causing in-progress reviews to be lost

**Tier:** [ ] Tier 1  [ ] Tier 2  [x] Tier 3

**Problem summary:**
When the agent runs a long review (multiple repos/tools), it tracks progress only
in the orchestrator's local memory and writes that progress to Redis a single
time, after every tool in the plan has finished. If the API process restarts
while a review is still in progress — a deploy, a crash, anything — none of the
work done so far is saved, and the review has to start completely over. The
code lives in `agent/orchestrator.py` (the `Orchestrator.run()` loop) and
`agent/memory/session_store.py` (the Redis-backed store it writes to). A
successful fix checkpoints each tool's result to Redis as soon as that tool
finishes, and on a fresh run checks that saved state first so completed tools
are skipped and only the unfinished ones actually re-execute.

**Scope reasoning ("Is this right for me?"):** This is a Tier 3 issue, a bigger
jump than the recommended Tier 1 starting point for a first contribution. I
chose it deliberately over the smaller Tier 1 health-check bug I'd originally
selected (#154). It's still tractable: it touches two files I could read in
full, the fix is a scoped change to *when* an existing Redis write happens
(not a new subsystem), and I verified my understanding by reading both files
end-to-end before writing any code. The estimated effort in the issue (7–10
hours) is real, mostly because a correct fix needs a second behavior beyond
checkpointing — actually skipping already-completed tools on resume — which
isn't obvious from the issue title alone.

**Branch name:** fix/47-persist-agent-progress-across-restarts

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** [4c0550b — docs: reproduce issue #47 and add Week 8 solution plan](https://github.com/LuisMend12/pathreview/commit/4c0550b)

**Reproduction summary:**
The fix for #47 (`cb5cc09`) and its regression tests
(`tests/unit/test_orchestrator.py`) were already committed on this branch
before I got to this week's reproduction step, so instead of writing a new
failing test from scratch I reproduced the original bug directly: I
temporarily replaced `agent/orchestrator.py` with its pre-fix version
(`git show cb5cc09^:agent/orchestrator.py`) and reran the existing
regression suite against it.

```
$ ./.venv/Scripts/python -m pytest tests/unit/test_orchestrator.py -q
...
FAILED tests/unit/test_orchestrator.py::TestOrchestratorCheckpointing::test_progress_is_checkpointed_after_each_tool_not_only_at_the_end
FAILED tests/unit/test_orchestrator.py::TestOrchestratorCheckpointing::test_restart_mid_review_resumes_without_rerunning_completed_tools
FAILED tests/unit/test_orchestrator.py::TestOrchestratorCheckpointing::test_previously_failed_tool_is_retried_on_resume
3 failed, 1 passed in 1.62s
```

The failure in `test_previously_failed_tool_is_retried_on_resume` shows the
bug concretely: `tools["tool_a"].execute.call_count` is `1` when it should
be `0` — the pre-fix orchestrator re-executes a tool that a prior run
(simulated via a seeded `session_store`) had already completed
successfully, because the loop in `Orchestrator.run()` never checked
existing session state per-tool and only persisted results once, after the
whole loop finished. I then restored `agent/orchestrator.py` via
`git checkout -- agent/orchestrator.py` and confirmed all 4 tests pass again
on the fixed version. This confirms the bug is real and pins it exactly to
the loop body in `Orchestrator.run()` described in `PLAN.md`.

**PLAN.md link:** [PLAN.md](PLAN.md)

**Walkthrough video (recommended):** _not recorded this week_

**Blockers or open questions:**
None blocking. Open question carried into Week 9: whether the "already
done" check should key off something more explicit than dict shape
(`success` key) so a future tool with a different result shape can't be
misread as complete — see Risks & unknowns in `PLAN.md`.

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
The implementation and its tests (`agent/orchestrator.py`,
`agent/memory/session_store.py`/`context_manager.py`,
`agent/error_handling.py`, `tests/unit/test_orchestrator.py`) were already
complete going into this week — all 5 sub-tasks from `PLAN.md`'s Plan
section are done. This week's work was verification and PR prep, not new
implementation: I ran `make test-unit` and confirmed the 53 failing tests
in the suite are pre-existing by diffing the failure list against the same
run on this branch's base commit (`54cc749`) — identical set, so this
change introduces no new failures. Same check for `ruff`/`mypy`: repo-wide
error counts went *down* (182→175 lint, 103→90 typecheck) because the fix
commit's type annotations closed 13 pre-existing mypy errors in the files
it touched, and none of the 5 changed files have any lint/typecheck/format
issues of their own.

**Next steps:**
PR #199 against `ascherj/pathreview` was already open from before Week 8
but had an empty template body — filled it in with the real summary,
changes, and testing/verification details this week. Still need to
request peer/mentor review in Slack.

**Blockers:**
None.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/199

**Branch:** `fix/47-persist-agent-progress-across-restarts`

**What you built:**
`Orchestrator.run()` now checkpoints each tool's result to Redis
immediately after it completes, instead of once at the very end of the
tool loop, and checks previously-saved session state before running each
tool so a restart resumes from where it left off — completed tools are
skipped, failed ones are retried.

**Tests added or updated:**
`tests/unit/test_orchestrator.py` (new, 4 tests): incremental
checkpointing after each tool, resume-after-restart across two
`Orchestrator` instances sharing one `session_store`, retry of a
previously-failed tool, and unchanged no-persistence behavior when
`session_store=None`.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
_(both confirmed with zero new failures relative to this branch's base
commit — see Check-in 1 for the verification method; full breakdown in the
PR description)_

**Draft PR feedback received from:** none yet — PR was already open
(not draft) from before Week 8; requesting peer/mentor review in Slack
this week
