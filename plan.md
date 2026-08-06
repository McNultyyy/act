# Splitting PR #6147 into reviewable pull requests

**Status:** PRs 1, 2 and 3 are open upstream. PRs 4, 5, 6 are gated on maintainer answers in #6147.
**Do not push this file to any PR branch** — it is a contributor working document, not something upstream wants.

| PR | branch | upstream | state |
|----|--------|----------|-------|
| 1 | `act-split/1-log-writer-races` | [#6153](https://github.com/nektos/act/pull/6153) | open |
| 2 | `act-split/2-step-command-dirs` | [#6154](https://github.com/nektos/act/pull/6154) | open, refs #2184 #2697 #2553 |
| 3 | `act-split/3-concurrency-queue-schema` | [#6152](https://github.com/nektos/act/pull/6152) | open, closes #6095 |
| 4 | `act-split/4-independent-workflow-runs` | — | not started |
| 5 | `act-split/5-concurrency-groups` | — | not started |
| 6 | `act-split/6-parallel-steps` | — | not started |

[#6147](https://github.com/nektos/act/pull/6147) is now a **draft** retitled "reference branch — being split, do not review", with a description linking the stack and asking the maintainers three questions: whether they want `concurrency` at all and in this shape, whether the PR 4 scheduling change is acceptable given it makes the known races in #6028/#6057/#2764 more likely, and whether the two features should be separate efforts. **Wait for those answers before building PRs 4–6.**

The blocking composite regression described below is **fixed** on `feat/concurrency-groups` in `b8f60ec`.

## Where the work currently lives

All of the implementation exists on `feat/concurrency-groups`, opened upstream as
[nektos/act#6147](https://github.com/nektos/act/pull/6147) (+3808 / −205 across 58 files). The fork remote is
`fork` → `github.com/McNultyyy/act`; `origin` is `nektos/act` and is not writable.

Source commits, oldest first:

| commit | contents |
|--------|----------|
| `b0edcfd` | concurrency groups, cancel-in-progress |
| `3a04185` | parallel steps: background / wait / wait-all / cancel / parallel |
| `946d5be` | merge of the two feature branches |
| `3b0e5f6` | fixes for the first code review (15 findings) |
| `c3f37ad` | `concurrency.queue`, case-insensitive groups, fixes for the second review (13 findings) |

That PR is **too large to review** and should not be merged as-is. It is being replaced by the stack below.

## The composite regression (FIXED in b8f60ec — kept for context)

`feat/concurrency-groups` **broke composite actions**. These three tests were green on `master`
and red on the branch.

```
go test ./pkg/runner/ -run 'TestRunEvent/uses-composite$|TestRunEvent/uses-nested-composite|TestRunEvent/composite-fail-with-output'
```

Cause: `useStepLogger` publishes per-step writers via `container.WithLogWriters(ctx, …)`, and
`docker_run.go` / `host_environment.go` prefer those over the environment's single writer slot. That
silently dead-codes `newCompositeCommandExecutor`'s `JobContainer.ReplaceLogWriter`
(`pkg/runner/action_composite.go:252`), so a composite action's inner steps have their `::set-output` /
`::save-state` parsed by the **outer** step's handler under the outer step id.

Two other capture sites are bypassed the same way and need checking:
`pkg/runner/expression.go:205` (`hashFiles` output capture) and `pkg/runner/run_context.go:586`
(node tool path detection). Both call `ReplaceLogWriter(hout, herr)` to capture into a buffer.

Fixed in `b8f60ec` by publishing all three through `container.WithLogWriters`, which also removes the last
non-nested swaps of the shared writer slot. Acceptance gate (all green):

```
go test ./pkg/runner/ -count=1 -run 'TestRunEvent$/uses-composite$|TestRunEvent$/uses-nested-composite$|TestRunEvent$/composite-fail-with-output$|TestRunEvent$/act-composite-env-test$|TestRunEvent$/do-not-leak-step-env-in-composite$|TestRunEvent$/outputs$'
```

**PR 6 must carry this fix**, since it is the PR that introduces the context writers. PR 2 avoids the
problem entirely by not shipping them — see the resequencing note.

Why it was missed: after the log-writer change only `-short` (skips integration) and named tests were run,
never the full docker `TestRunEvent`. Any PR touching log writers must run that suite.

## Branches

Created off `master`, all currently empty:

| branch | PR |
|--------|-----|
| `act-split/1-log-writer-races` | 1 |
| `act-split/2-step-command-dirs` | 2 |
| `act-split/3-concurrency-queue-schema` | 3 |
| `act-split/4-independent-workflow-runs` | 4 |
| `act-split/5-concurrency-groups` | 5 |
| `act-split/6-parallel-steps` | 6 |
| `act-split/plan` | this document |

PRs 1–4 are mutually independent and branch off `master`. Before developing PR 5, merge branches 3 and 4
into it; before PR 6, merge 1 and 2. Example:

```
git checkout act-split/5-concurrency-groups
git merge act-split/3-concurrency-queue-schema act-split/4-independent-workflow-runs
```

## The stack

| # | PR | ~lines | Closes | Refs | Depends on |
|---|-----|--------|--------|------|------------|
| 1 | Guard log-writer slots and mask list | 120 | — | #2199 | — |
| 2 | Per-step-execution runner file command dirs | 280 | — | #2184 #2697 #2553 #2273 | — |
| 3 | Accept `concurrency.queue` in the schema | 230 | **#6095** | #6086 | — |
| 4 | Execute each workflow as an independent run | 300 | — | #158 #2492 #2732 #2756 | — |
| 5 | Workflow and job level `concurrency` groups | 1750 | — | #6095 #2732 | 3, 4 |
| 6 | Parallel steps | 1400 | **#6124** | #2287 | 1, 2 |

**Order:** put 1, 2 and 3 up simultaneously. Then 4. Then 5 and 6, which are independent of each other and
can be reviewed in parallel.

---

### PR 1 — `fix: guard the shared log-writer slots and mask list against concurrent access`

**Scope.** Pure data-race hardening of code already reachable today; no behaviour change.
- `logWriterMutex` around `containerReference.ReplaceLogWriter`, plus a `getLogWriters()` accessor used by
  `waitForCommand` and `attach`.
- `stdoutMutex` / `getStdOut()` on `HostEnvironment`.
- `ptyWriter.AutoStop` becomes `atomic.Bool` — it is written on the exec goroutine after `copyPtyOutput`
  has already started reading it, a genuine pre-existing race on `--self-hosted`.
- Package-level `masksMutex` so `valueMasker` reads a copy while `RunContext.AddMask` appends.
- `RunContext.masksSnapshot()`, used by `newCompositeRunContext` / `execAsComposite` instead of aliasing
  and appending to the parent's slice.

**Explicitly excluded:** the context-carried log writers (`container.WithLogWriters`). Those belong to PR 6.

**Files:** `pkg/container/docker_run.go`, `pkg/container/host_environment.go`,
`pkg/container/host_environment_test.go`, `pkg/runner/logger.go`, `pkg/runner/run_context.go`,
`pkg/runner/action_composite.go`

**Standalone value.** Three real races fixed with no semantic change. Maintainers merge small race fixes readily.

**Risks.**
- `masksMutex` is a package-level global guarding per-`RunContext` slices — coarse. Expect a request to move
  it onto `RunContext`.
- `valueMasker` copies the mask slice per log line — an allocation in a hot path. Mitigate by only copying
  when `len(*masks) > 0`, or hold the RLock across the `ReplaceAll` loop.
- Does **not** fix #2199's most recent stack (map iteration over shared `model` structs in
  `NewExpressionEvaluatorWithEnv`). Do not claim closure.

**Tests.** `go test -race ./pkg/container/... ./pkg/runner/...`. Add a targeted unit test writing to a
`ptyWriter` from one goroutine while flipping `AutoStop` from another — fails under `-race` on master. The
mask and writer-slot races have no deterministic test; say so in the PR body rather than inventing one.

---

### PR 2 — `fix: give each step execution its own runner file command directory`

**Scope.** Replaces the shared `workflow/outputcmd.txt`, `statecmd.txt`, `pathcmd.txt`, `envs.txt`,
`SUMMARY.md` with `workflow/cmds-<seq>-<stage>/…` per step execution (`stepFileCommandSeq atomic.Int64`),
so a step never re-reads the previous step's leftover env-file commands. Routes results by explicit step id:
`setOutput` → `setOutputForStep`, `saveState` → `saveStateForStep`, `commandHandler(ctx)` →
`stepCommandHandler(ctx, stepID)` at the three call sites that know their step. Adds
`RunContext.parentStepID` so `getScriptName` no longer derives composite script names from the mutable
`CurrentStep`. Test-mock churn: a `matchCmdFile()` helper so the four step unit tests match the new paths.

**Keeps `JobContainer.ReplaceLogWriter` in `useStepLogger`** — see risk.

**Files:** `pkg/runner/step.go`, `command.go`, `action.go`, `step_docker.go`, `step_run.go`,
`run_context.go`, `action_composite.go`, `container_mock_test.go`, `step_action_local_test.go`,
`step_action_remote_test.go`, `step_docker_test.go`, `step_run_test.go`, `runner_test.go`, plus a new
`testdata/composite-undeclared-outputs/` fixture.

**Standalone value.** This is the actual fix for the composite-output leak reported three times
(#2184, #2697, #2553 — the last two by a maintainer). Independent of every other slice and the
highest-value small bugfix in the branch.

**Risks.**
- **Deliberate deviation from the branch:** ship this *without* the ctx-carried log writers. On the branch
  they arrive together and unfixed, which is the regression above.
- In-container paths of `GITHUB_OUTPUT` / `GITHUB_ENV` / `GITHUB_STEP_SUMMARY` change — anything
  hard-coding them breaks.
- Per-execution directories are never cleaned up and accumulate for the job's lifetime.
- `$GITHUB_STEP_SUMMARY` becomes per step *execution* rather than per step id.

**Tests.** New fixture asserting an undeclared composite inner-step output is **not** visible on the outer
step, and that a declared output mapping `steps.<inner>.outputs.<name>` still resolves — for plain and
nested composites, via both `$GITHUB_OUTPUT` and `::set-output`. This fixture does not exist on the branch
and a maintainer will want it before closing a fidelity bug. Then `TestRunEvent` under docker:
`uses-composite`, `composite-fail-with-output`, `uses-nested-composite`, `act-composite-env-test`,
`do-not-leak-step-env-in-composite`, `outputs` must all pass.

---

### PR 3 — `feat: accept concurrency.queue in the workflow schema` ← **start here**

**Scope.** Schema-only sync with GitHub plus the model types it needs. Adds a `queue` property to
`concurrency-mapping` and a `concurrency-queue` definition with `allowed-values: [single, max]` (verified
against `actions/languageservices` `workflow-v1.0.json`, which defines `queue` on `concurrency-mapping`).
Because `workflow-concurrency` and `job-concurrency` both one-of into `concurrency-mapping`, workflow-, job-
and reusable-workflow-level blocks are all covered. Adds `model.Concurrency{Group, CancelInProgress, Queue}`,
`parseConcurrency` (scalar shorthand + mapping), `Workflow.Concurrency()`, `Job.Concurrency()` and
`validateConcurrency()` wired into both `Workflow.UnmarshalYAML` and `WorkflowStrict.UnmarshalYAML`.
**Runtime behaviour unchanged** — act still ignores concurrency, exactly as on master.

**Files:** `pkg/schema/workflow_schema.json`, `pkg/model/workflow.go`, `pkg/model/workflow_test.go`

**Standalone value.** Closes **#6095** verbatim: `act -l` currently dies with `Unknown Property queue` on a
valid workflow. Roughly ten lines of schema. Same shape as #6086 / #2766 / #2621 and the single most
obviously-mergeable change in the branch.

**Risks.**
- `validateConcurrency` makes act **stricter than the shared schema**: `queue: max` + `cancel-in-progress: true`
  fails to parse, where GitHub accepts it in the schema and enforces at run time. The #6095 reporter asked
  for exactly this, but be ready to downgrade to a warning.
- `parseConcurrency` returns nil when the group is empty, so `concurrency: {cancel-in-progress: true}` with
  no group is silently ignored rather than rejected.
- The enum is case-sensitive, so `queue: Max` is rejected. Matches SchemaStore/actionlint; worth a line in
  the PR body.

**Tests.** Table-driven in `pkg/model/workflow_test.go`: scalar shorthand, mapping form, `single`/`max`,
expression values (`${{ }}` must bypass both the enum and the combination check), job-level-only
concurrency, the rejected combination at workflow and job level, `--strict` parity.
`go test ./pkg/model/... ./pkg/schema/...`

---

### PR 4 — `refactor: execute each workflow as an independent run instead of one global stage pipeline`

**The behaviour-changing PR. Must not be folded into PR 5.**

**Scope.** Today `NewPlanExecutor` walks `plan.Stages` serially and runs each stage's jobs with
`NewParallelExecutor(GetConcurrentJobs())`, making the global stage index a barrier across *unrelated*
workflows. This regroups the plan's runs per `*model.Workflow`, runs workflows in parallel, keeps each
workflow's own stages serial (so `needs` semantics are untouched), and replaces per-stage parallelism with a
plan-wide `jobSlots chan struct{}` semaphore of size `GetConcurrentJobs()` taken inside the job executor via
`RunContext.withJobSlot`. Adds `RunContext.jobSlots`. **Nothing** about concurrency groups, cancellation or
`runState` belongs here.

**Files:** `pkg/runner/runner.go`, `pkg/runner/run_context.go`, `pkg/runner/runner_test.go`

**Standalone value.** Two things evaluable without caring about concurrency: an unrelated workflow's slow job
no longer blocks another workflow's jobs from starting (a partial answer to #158 and the concrete scenario in
#2492); and `--concurrent-jobs` becomes an explicit plan-wide semaphore rather than a per-stage argument,
which is the prerequisite for a job ever waiting without occupying a slot. **Honest framing required:**
intra-workflow stages stay serial, so #158's actual ask (drop `model.Stage`, start a job the moment its own
`needs` are satisfied) is *not* delivered.

**Risks — HIGH.** The one PR that changes what every act user sees even using none of these features.
Enumerate all of this in the PR body:
- Job start order and log interleaving change for any invocation planning more than one workflow file —
  which is the default `act` invocation.
- **Must fix before upstreaming:** the branch shares `maxJobNameLen` as a `*int` written from concurrently
  running workflow-run goroutines with no synchronisation — a straight data race and non-deterministic
  job-name padding. Compute the max over all runs up front, before any executor starts.
- Real parallelism goes up, making known unfixed races more likely to fire: GoGitActionCache (#6028),
  LocalRepositoryCache (#6057), shared `RawRunsOn` evaluation (#2764 / #5971). This is the strongest
  argument against merging and should be stated, not hidden — arguably this PR should wait until those are
  fixed.
- Reusable workflows call `NewPlanExecutor` again on the shared config, minting a fresh `jobSlots`, so
  `--concurrent-jobs` is still not a true global cap once reusable workflows are involved. Do not overclaim.
- `handleFailure(plan)` now runs after a parallel join rather than a serial pipeline, so error aggregation
  across workflows changes.

**Tests.** Needs tests it does not currently have: two independent workflows demonstrably interleave;
`--concurrent-jobs 1` still serializes everything; a workflow's stage N+1 job never starts before all of its
own stage N jobs finish. Then the full docker `TestRunEvent`, plus `go test -race ./pkg/runner/...`
specifically for the `maxJobNameLen` race.

---

### PR 5 — `feat: support workflow and job level concurrency groups with cancel-in-progress and queue`

**Scope.** The runtime for the syntax PR 3 taught act to parse. New `pkg/runner/concurrency.go`:
a `concurrencyManager` (one holder per group, FIFO pending queue, case-insensitive keys, `queue: single`
supersedes the pending request, `queue: max` admits up to 100 and cancels arrivals once full);
`evaluateConcurrency`; `workflowRunState` for run-scoped cancellation; a context-carried set of held groups
so a reusable workflow does not deadlock on a group its own caller holds; and `RunContext.withConcurrency`
wrapping the job executor inside `RunContext.Executor()` so only jobs that actually run take part. Wires
workflow-level groups into PR 4's `newWorkflowRunExecutor`. Cancelled runs and jobs get result `cancelled`
without failing the plan. The manager lives on `Config`, so it is shared with reusable-workflow runners.

**Files:** `pkg/runner/concurrency.go`, `concurrency_test.go`, `runner.go`, `run_context.go`,
`runner_test.go`, plus `testdata/concurrency*/` fixtures and
`testdata/.github/workflows/local-reusable-concurrency.yml`

**Standalone value.** `concurrency` is currently one of act's documented "parsed but ignored" keys; this
makes it behave like GitHub. Also makes a job waiting on a group release its `--concurrent-jobs` slot.
Large, but the manager plus its test file is one cohesive unit — splitting into job-level then
workflow-level would leave the second PR as a ~40-line diff on top of a manager nobody could review in
isolation.

**Risks.** Medium-high, mostly because it turns a no-op key into real scheduling.
- Users who already have `concurrency:` blocks will suddenly see jobs serialize and get cancelled where act
  previously ran them all. GitHub-correct but visible — call it out, consider a release note.
- cancel-in-progress is cooperative (a `jobCancelCtx` the job observes), so a container may take up to the
  graceful-stop timeout to die.
- Deadlock avoidance is a context-carried set of held group names, so a group reused between a caller and a
  nested reusable workflow is silently *skipped* rather than queued. Deliberate, but act-specific.
- The group's `queue` policy is taken from whichever request created the group; later disagreeing requests
  only get a warning. **act policy, not GitHub behaviour** — document it.
- The 100-deep FIFO is exercised only by unit tests, never end to end.

**Tests.** Keep the unit tests in `concurrency_test.go` (serialization, superseding, case-insensitive groups,
FIFO under `queue: max`, overflow cancellation, abandoned-holder promotion, cancel-ctx aborts, parse-time
validation). Keep the `concurrency` fixture in the docker `TestRunEvent` table. Trim the `*-windows` fixture
duplicates to only those the test table references. Run `go test -race ./pkg/runner/ -run 'Concurrency'` —
for a manager like this, the race-detector run *is* the review evidence.

---

### PR 6 — `feat: run steps in parallel with background, wait, wait-all, cancel and parallel`

**Scope.** The step-level parallelism feature plus the thread-safety rework it requires, which cannot stand
alone. Model + schema: `background` on run-step and regular-step; `wait-step` / `wait-all-step` /
`cancel-step` / `parallel-step` in `steps-item`; matching `StepType` values and accessors. Runtime: new
`pkg/runner/step_background.go` with a 10-slot registry, `expandParallelStepGroups` desugaring a `parallel:`
block into background steps plus a synthesized `wait`, wait/cancel by step id, first-wait-takes-the-error
semantics, cancelled steps not failing the job, `cancelRemaining()` at job end; `job_executor.go` routes
background and control steps. Thread-safety: `pkg/container/log_writer.go` with `WithLogWriters` /
`LogWriters` honoured by `docker_run.go` and `host_environment.go`; `RunContext.stepStateMu` and
`stateDataMu` with `envSnapshot` / `stepsSnapshot` feeding the expression evaluators; `setCurrentStep` /
`currentStep`; the composite `ExtraPath` merge instead of overwrite.

**Files:** `pkg/model/workflow.go`, `workflow_test.go`, `pkg/schema/workflow_schema.json`,
`pkg/runner/step_background.go`, `step_background_test.go`, `job_executor.go`,
`pkg/container/log_writer.go`, `docker_run.go`, `host_environment.go`, `pkg/runner/action_composite.go`,
`step.go`, `run_context.go`, `expression.go`, `command.go`, `runner_test.go`, plus `testdata/background-steps*/`

**Standalone value.** Closes **#6124**. The reporter's real workflow (`background: true` with `env:`, a
3-step `parallel:` block, a bare `wait-all:`) parses with zero errors and executes with correct ordering and
real overlap. This is genuine upstream GitHub syntax, not an act extension: `actions/languageservices`
`workflow-v1.0.json` defines `steps-item` as the same six-member one-of, `background` on both step kinds,
`wait` as string-or-sequence, `wait-all` as null-or-boolean. Composite actions correctly still reject these
keys because GitHub's own `action-v1.0.json` does too — say so in the PR body so it is not read as a gap.

**Risks — highest complexity of the stack, and it carries the confirmed regression.**
- **MUST FIX:** `newCompositeCommandExecutor` still installs its handler via
  `JobContainer.ReplaceLogWriter`, which this PR's ctx-writer preference silently dead-codes. Publish the
  composite handler through `container.WithLogWriters` too, then re-run the three composite tests.
- **MUST FIX:** `expandParallelStepGroups` mutates shared `*model.Step` pointers (`child.ID`,
  `child.Background = "true"`) that matrix `RunContext`s share. Empirically collision-free because the ids
  are deterministic, but a reviewer will flag it. Copy the step.
- Schema deltas from upstream to close: `cancel-step` is missing `continue-on-error` (GitHub has it);
  act's `parallel-steps.item-type` is a narrower `parallel-steps-item` where GitHub uses `steps-item`, so
  nested `parallel` and `wait`/`cancel` inside a group are wrongly rejected; act's `parallel-step` permits
  `name`/`id` where GitHub's permits only `parallel` — but `expandParallelStepGroups` uses the group's id
  for the synthesized wait step, so removing it needs a different id scheme. Rename `wait-targets` /
  `wait-all-value` to GitHub's `step-wait-target` / `step-wait-all-value` so future schema syncs diff cleanly.
- Cosmetic: the "queued until one of the 10 slots is free" warning fires spuriously because `queued()`
  races the launch goroutine.
- Do **not** claim this fixes #2287 — a shell `&` inside a `run` step still dies with the exec; this is a
  different, opt-in mechanism requiring a workflow rewrite.

**Tests.** Keep `TestExpandParallelStepGroups`, `TestRunBackgroundSteps`, `TestRunBackgroundStepFailure`.
Keep `background-steps-container` in the docker `TestRunEvent` table. Add a matrix fixture exercising two
`parallel` groups plus a loose background step across three matrix entries, to lock in that auto-assigned
ids stay collision-free. Re-run the three composite tests as the acceptance gate for the log-writer change,
and `go test -race ./pkg/runner/...` for the locking.

---

## Sequencing notes

PRs 1–3 are independent of each other and everything else; they can go up simultaneously in any order. They
are first because each is small, each is a bug fix, and none changes behaviour for anyone not already
hitting the bug.

PR 4 is the pivot and is deliberately isolated. Its body must carry its own justification — cross-workflow
barrier removal and `--concurrent-jobs` becoming a real semaphore — and must not borrow motivation from
concurrency groups.

**If PR 4 is rejected or stalls, PR 5 is still shippable in reduced form: job-level `concurrency` only.**
`withConcurrency` already tolerates `runState == nil` and `jobSlots == nil`, so it works on master's stage
executor. The cost is that a job queued on a group occupies a `--concurrent-jobs` slot while waiting —
wasted parallelism, no deadlock, since jobs within a stage are independent and the reusable-workflow
self-deadlock case is handled by the held-group context. Workflow-level groups genuinely require PR 4,
because a run must hold its group from its first to its last job. **Do not attempt workflow-level groups on
top of the global stage pipeline.**

PR 6 depends on PR 2 (background steps sharing one `workflow/outputcmd.txt` would corrupt each other's
outputs) and on PR 1 (it makes the writer-slot races reachable in normal operation). It is independent of
PRs 3–5 and could be reviewed in parallel by someone else.

**The deliberate deviation worth stating to reviewers up front:** PR 2 ships the per-step file-command
directories *without* the context-carried log writers, and PR 6 ships those writers *together with* the
`newCompositeCommandExecutor` fix. On the branch these arrived together and un-fixed, which is why the three
composite-output issues currently fail verification and three existing tests go red. Splitting them this way
is what makes PR 2 mergeable on its own and PR 6 correct.

**Size honesty.** PRs 5 and 6 are ~1750 and ~1400 lines, dominated by their own test files and fixtures.
Splitting PR 5 into manager-then-wiring, or PR 6 into locking-then-feature, would produce a leading PR that
adds unreachable code and a trailing PR that is a 40-line diff — neither half stands alone.

## Must not ship as-is

1. **`maxJobNameLen *int`** in PR 4 — written from concurrent workflow-run goroutines with no
   synchronisation. Compute it up front instead.
2. **`expandParallelStepGroups` mutating shared `*model.Step` pointers** — copy the step.
3. **`newCompositeCommandExecutor` using `ReplaceLogWriter`** once ctx writers take precedence. Confirmed
   cause of three red tests. Either the ctx-writer change does not ship (PR 2) or the composite handler
   moves to `WithLogWriters` in the same PR (PR 6).
4. **The ten `*-windows` duplicate testdata fixtures** — ship only those the test table references.
5. **act's narrower `parallel-steps-item`** — match GitHub's `steps-item`. Same for `cancel-step`'s missing
   `continue-on-error` and the `wait-targets` / `wait-all-value` naming.

## Claims that must not appear in a PR body

Checked and do **not** hold:

- **#2184, #2697, #2553 as "Closes".** The scoped PR 2 removes the reason all three verdicts failed, so it
  plausibly does close them — but re-verify against the reporters' repros after the split, and only once the
  regression fixture exists. Reference them; do not close them.
- **#2199 as closed.** Its most recent stack is map iteration over shared `model` structs in
  `NewExpressionEvaluatorWithEnv`, which nothing here locks.
- **#2287 as closed.** A shell `&` in a `run` step still dies with the exec.
- **#158 or #2492 as closed.** Stages inside a single workflow remain serial, so #2492's exact scenario
  behaves identically to master.
- **#6028, #6057, #2764, #5971, #6012 as addressed.** None of those files is touched, and PR 4 makes several
  of them *more* likely to fire.

Verified as genuine closes: **#6095** (PR 3) and **#6124** (PR 6).

## Policy questions for maintainers

Raise these rather than deciding unilaterally:

1. The parse-time hard error on `queue: max` + `cancel-in-progress: true` makes act stricter than the shared
   schema. The #6095 reporter asked for it; be prepared to downgrade to a warning.
2. "The group's queue policy wins for later arrivals" is act-invented with no GitHub equivalent. Keep it —
   the alternative lets a workflow omitting `queue: max` discard a queue admitted under it — but document it
   explicitly rather than letting it read as fidelity.
3. `masksMutex` as a package-level global, and the per-log-line slice copy in `valueMasker`. Fine as an
   interim fix; expect a request to move the lock onto `RunContext` and avoid the allocation when there are
   no masks.

## Resuming

```
# see the full implementation
git checkout feat/concurrency-groups

# reproduce the blocking regression (needs docker)
go test ./pkg/runner/ -run 'TestRunEvent/uses-composite$|TestRunEvent/uses-nested-composite|TestRunEvent/composite-fail-with-output'

# start the stack
git checkout act-split/3-concurrency-queue-schema
```

Suggested first actions:
1. Fix the composite regression on `feat/concurrency-groups` so it is not left broken.
2. Build PR 3 and open it — smallest, closes a verified issue, earns standing with the maintainers.
3. Before investing in PRs 4–6, comment on #6147 describing the stack and asking the maintainers how they
   want it. They may have their own design in mind for `concurrency`, which their docs list as "planned".

## Reality check on getting these merged (researched 2026-08-06)

**nektos/act is effectively dormant.** Last merged PR of any kind: #6089, 2026-05-13 (85 days). Last human
commit on master: 2026-03-26. Merge counts: 150 in 2024, 78 in 2025, 7 in 2026. Since 2026-04-06, 41
external PRs have been opened and received zero reviews, zero maintainer comments and zero merges.
ChristopherHX, panekj and cplee are all active on GitHub elsewhere but have no events in this repo.

**Two hard blockers that no amount of etiquette fixes:**

1. *Fork CI approval.* No fork run has been approved since 2026-04-05. 38 runs sit at `action_required`
   across ~12 contributors; 16 of them are ours. Until someone clicks approve, our PRs have literally zero
   status checks, so every mergify `check-success=` condition is unsatisfiable by construction.
   These runs age out of the ~90-day retention window around **early November 2026** — push a trivial
   commit before then or the PRs end up permanently CI-less, like #6096 and #6099 already are.
2. *`lint` is red repo-wide.* MegaLinter's project-scoped grype (44 findings) and osv-scanner (38) fail on
   master over dependency CVEs. `VALIDATE_ALL_CODEBASE: false` does not exempt project-mode linters, so
   every PR inherits it. `check-success=lint` is currently unsatisfiable for everyone, dependabot included.
   The obvious fix is circular: the x/crypto and x/net bumps that would clear it themselves broke
   `test-linux` on 2026-07-03 and 2026-07-10.

**Merge arithmetic.** Empirically an outside PR needs two write-access approvals, or one from `cplee`.
An approval is halfway, not done: #6055 (approved, all checks green) has sat 4 months, #6007 6 months,
#5895 12 months. Peer approvals from other contributors do not count — mergify only counts admin/write/
maintain permission. `cplee` is the only person who direct-merges past mergify (#6042: 58 minutes, zero
reviews).

**Codecov is NOT a blocker** — `codecov/patch` is `informational: true` so it cannot fail, and tokenless
fork uploads are proven working. Do not spend effort on patch coverage.

**What does not work:** "any update?" comments have a 0% response rate in 2026 (#6039 collected three, all
ignored). GitHub Discussions has a 0% hit rate for this exact question (#6118, 48 days, no replies). There
is no Discord/Slack/Gitter — #2678 deliberately removed the last one.

**What does work:** cross-references from an ISSUE a maintainer is already triaging. That is the only 2026
path from cold PR to approved PR — #6055 sat untouched until panekj, triaging #6057, cross-referenced it,
then approved both the workflow run and the PR within 24 hours.

**Done on 2026-08-06:** posted a real reproduction of the composite-output leak on #2553 (ChristopherHX's
own 2024 bug, previously bodyless), cross-referenced #2184 and #2697, and posted the `-race` findings on
#6057 (the tracker panekj asked be kept open). Also corrected a duplicate-PR problem — see below.

**Duplicate-PR correction.** #6152 duplicated #6096 by `xenjke`, who filed issue #6095 and opened their PR
one minute later; we opened ours months after and announced it on their issue without acknowledging theirs.
Fixed: #6152 retitled so it no longer claims to close #6095, now points at #6096 and only carries the
`pkg/model` half; apology and offer to fold in posted on #6096; correction posted on #6095.
**Check open PRs, not just issues, before building anything further.**

**Not done deliberately:** no ping to `cplee`. The research suggests leading with the repo-wide `lint`
breakage would get attention (his direct merges have a demonstrated sub-hour response to supply-chain
breakage), but that is an escalation to the project founder and should be a considered decision, not a
reflex.