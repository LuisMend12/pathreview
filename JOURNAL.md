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

## Week 10 — Iteration & reflection

### Reviewer feedback

**Feedback received:** [ ] Yes  [x] No — still awaiting review

**Summary of feedback:**
No reviewer comments arrived on PR #199 by the end of Week 10. Summer
2026 cohort note: reviewer feedback is not a feature this term.

**How you responded:**
N/A — no feedback to respond to.

---

### Reflection

**What was harder than you expected?**
The mypy pre-commit hook was the unexpected blocker. The core fix to
`Orchestrator.run()` is about 10 lines — move `session_store.set()` inside
the loop, add a per-tool `already_done` check. But before that commit
could land, the pre-commit hook ran mypy across every file that
`orchestrator.py` imports, and three of them (`error_handling.py`,
`context_manager.py`, `session_store.py`) had missing return-type and
parameter annotations that caused mypy to fail. None of those annotation
gaps were related to the bug I was fixing — they were pre-existing tech
debt. Tracking down exactly which annotations were needed, across three
files I hadn't planned to touch, took longer than the fix itself.

**What did you learn about working in a large codebase?**
The dependency graph matters as much as the code you're changing. In a
project I own, I touch a file and push. Here, touching `orchestrator.py`
meant understanding everything it imports, because the CI pipeline treats
the whole import chain as a unit. I also learned to read the existing
tests before writing new ones — `tests/unit/` had a consistent fixture
pattern (in-memory fakes, no external services) that I needed to match so
my tests would be collected by `make test-unit` with the right markers.
Skimming the existing test files first saved me from writing tests that
passed locally but got skipped in CI.

**How did AI tools help — and where did they fall short?**
AI was most useful for orientation: tracing the call graph from
`Orchestrator.run()` through `session_store.get/set()` and understanding
the JSON round-trip contract without reading every line of every file.
It was also useful for drafting the `FakeSessionStore` — I described what
I needed (in-memory, records every `set()` call, round-trips through JSON)
and got a solid starting structure I could verify and adjust. Where AI
fell short: it couldn't tell me which specific mypy errors would fire until
I actually ran `mypy` locally. It gave me plausible annotation patterns,
but the exact error messages — "Missing return type annotation for public
function" on line N — required running the tool. AI gave me the shape;
local execution gave me the specifics.

**What would you do differently if you started over?**
Run `make check` immediately after setting up the environment, before
reading any code. I'd have seen the baseline — 103 mypy errors, 182 lint
warnings repo-wide — and known upfront that some of those pre-existing
errors were in files my change would touch. Instead I discovered the
annotation gaps only when the pre-commit hook blocked my first commit
attempt. Knowing the baseline lets you plan which adjacent files need
cleanup before you commit; not knowing it turns pre-existing debt into a
surprise blocker.

**What are you most proud of from this module?**
The `FakeSessionStore` design in `tests/unit/test_orchestrator.py`. It
would have been easy to mock `session_store.get` and `session_store.set`
directly with `unittest.mock.Mock`, but that wouldn't catch bugs that only
appear after a real JSON serialize/deserialize cycle (a `datetime` in a
tool result would pass a Mock-based test and silently fail in production).
The fake instead round-trips every `set()` call through `json.loads(json.dumps(data))`
and records each snapshot. That's the same guarantee the real Redis store
provides, which means the tests are actually testing the checkpointing
contract rather than just asserting that certain methods were called.
