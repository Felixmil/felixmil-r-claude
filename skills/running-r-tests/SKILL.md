---
name: running-r-tests
description: Use whenever the user asks to run R package tests, triage test failures, verify a fix, or decide between running tests inline versus dispatching the r-test-runner subagent. Activate on phrases like "run the tests", "are tests passing", "fix this failure", "test_file", "test_local", "devtools::test", "rerun the failing test", and on any reference to a triage / red-to-green loop in an R package.
---

# Running R package tests

This skill covers **test execution and failure triage** for R packages. For writing or modifying test code, use the `r-lib:testing-r-packages` skill instead — they compose: that one tells you *how to write a test*, this one tells you *how to run and triage tests*.

## Dispatch by scope

**Always run the narrowest scope that addresses the question.** Three tiers:

- **Single test** (one `test_that()` or `it()` block) → run inline with `Rscript -e 'devtools::load_all(); testthat::test_file("<path>", desc = "<exact description>")'`. Use this whenever you know the specific failing test from a prior run — re-running the whole file wastes time.
- **Single test file** → run inline with `Rscript -e 'devtools::load_all(); testthat::test_file("<path>")'`. Output is small enough to read directly; subagent dispatch just adds latency.
- **Multiple test files or the full suite** → dispatch the `r-test-runner` subagent. **The exact dispatch shape is fixed** — see the canonical call below. Do not split it, do not omit `run_in_background`, do not run `Rscript` inline instead.

### Canonical subagent call

There is exactly **one** correct way to invoke `r-test-runner` from this skill. All three properties are required as a single atomic gesture:

```
Agent tool with:
  subagent_type:      "r-test-runner"
  run_in_background:  true
  prompt:             "<dispatch instructions, including LOGDIR=<resolved-path>>"
```

Forgetting any one of these properties is a bug:

- **Wrong tool** (`Bash` instead of `Agent`) → floods the parent context with raw `Rscript` output and bypasses the agent's SummaryReporter contract that workflow step 4 depends on. Always use the Agent tool for multi-file or full-suite runs.
- **Missing `run_in_background: true`** → the subagent runs in foreground and you watch its full transcript. The summary still lands, but the context cost is wasted. Always background.
- **Missing `subagent_type: "r-test-runner"`** → you'd dispatch the wrong agent (or a generic one). Always pin the type.

The only exceptions to this whole rule: (a) the user explicitly asks for raw inline output, or (b) the `Agent` tool is unavailable.

### Why `devtools::load_all()` first for inline runs

`testthat::test_file()` only sources the test file, not `R/`. Without `load_all()`, package functions aren't in memory and the test fails with "could not find function". The subagent is exempt — `testthat::test_local()` source-loads the package on its own.

### About the subagent

The `r-test-runner` subagent writes logs and summaries under a configurable directory — default `./r-tests-runner/`, overridable per-project or globally by setting `R_TEST_RUNNER_LOG_DIR` in `settings.json`'s `env` block. The directory gets a self-ignoring `.gitignore` on first run, so it's safe to leave untracked. Each dispatch produces `<LOGDIR>/<timestamp>.log` plus a `<LOGDIR>/0_latest.log` symlink (the `0_` prefix keeps it at the top of `ls`); the user can `tail -f <LOGDIR>/0_latest.log` to watch a run in progress. The reply always includes the resolved log path on a `**Log:**` line, so just read that — don't assume the default.

#### Resolving the log dir before dispatch

**The subagent does not look up the env var itself** — subagent shells may not see the parent's `env`, and even if they did, an `echo` for the lookup would hit a permission gap. The parent (you) resolves it once in the parent shell and passes it in the dispatch prompt:

1. Run `echo "${R_TEST_RUNNER_LOG_DIR:-./r-tests-runner}"` in the parent's Bash to get the resolved path. (Parent's shell sees `env` from `settings.json`.)
2. Include the line `LOGDIR=<resolved-path>` somewhere in the dispatch prompt to the `r-test-runner` subagent.
3. If you forget, the subagent defaults to `./r-tests-runner` — same as if no env var were set.

The subagent uses `testthat::SummaryReporter`, which emits the failing `test_that()` description in each `Failure ('file:line:col'): <description>` header. Use that exact description verbatim as the `desc =` argument when you re-run the failing test inline (workflow step 4 below).

## Triage & regression workflow

**Use this loop whenever you're triaging failures or verifying a fix across a package** — i.e. you don't yet know if the suite is green, or you've just changed code and need to confirm nothing regressed. Skip it for narrow questions ("does test-foo pass?" → just run that file) and for TDD on a single new test (write → run that one test → iterate, no full-suite dance until the feature is done).

1. **Run the full suite first via the `r-test-runner` subagent** — use the `Agent` tool (`subagent_type: "r-test-runner"`, `run_in_background: true`). Do **not** call `Rscript -e 'testthat::test_local(...)'` yourself; the subagent is the canonical entry point per the hard rule above. **Do not** assume "full suite" is wasteful — the subagent is built to make this cheap, and its summary lists every failing file with the failing test names.
2. **Identify a failure to fix** from the summary.
3. **Implement the fix** in the relevant source file.
4. **Re-run only the failing test inline** with `Rscript -e 'devtools::load_all(); testthat::test_file("<path>", desc = "<exact description from the summary>")'`. This is the fastest feedback loop. Only fall back to running the whole file (drop the `desc` argument) when the failure is order-dependent or the description is ambiguous.
5. **When that test passes, re-run its file** to catch nearby regressions, then re-run the full suite via the subagent to catch regressions elsewhere.
6. Repeat steps 2–5 until all pass.

Never run a single failing file in a loop while ignoring the rest of the suite — you'll miss regressions. Never run only the suite and never drill into individual files — you'll waste turns re-reading a long summary.

If the user explicitly asks for the subagent on a single file, or for raw output on a multi-file run, honor that override.
