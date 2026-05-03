---
name: running-r-tests
description: Use whenever the user asks to run R package tests, triage test failures, verify a fix, or decide between running tests inline versus dispatching the r-test-runner subagent. Activate on phrases like "run the tests", "are tests passing", "fix this failure", "test_file", "test_local", "devtools::test", "rerun the failing test", and on any reference to a triage / red-to-green loop in an R package.
---

# Running R package tests

This skill covers **test execution and failure triage** for R packages. For writing tests, use the `r-lib:testing-r-packages` skill instead — that one tells you *how to write a test*, this one tells you *how to run and triage tests*.

## Dispatch by scope

Always run the narrowest scope that addresses the question:

- **Single test** (one `test_that()` block) → run inline: `Rscript -e 'devtools::load_all(); testthat::test_file("<path>", desc = "<exact description>")'`. Use this when you already know which test to re-run.
- **Single file** → run inline: `Rscript -e 'devtools::load_all(); testthat::test_file("<path>")'`. Output is small enough to read directly.
- **Multiple files or the full suite** → dispatch the `r-test-runner` subagent (Agent tool, `subagent_type: "r-test-runner"`, foreground — the user needs to approve setup commands on first run).

`devtools::load_all()` is required for inline `test_file()` calls because `test_file()` doesn't source `R/`; without it, tests fail with "could not find function". The subagent's `test_local()` source-loads the package on its own, so it doesn't need `load_all()`.

### About the subagent

The subagent writes logs under `<LOGDIR>` (default `./r-tests-runner/`, override with `R_TEST_RUNNER_LOG_DIR` in `settings.json`'s `env` block). It resolves the env var itself via `Sys.getenv` — you don't need to pre-resolve or pass `LOGDIR=` in the dispatch prompt. The dispatch reply includes the resolved log path on a `**Log:**` line. The user can `tail -f <LOGDIR>/0_latest.log` to watch a run.

The subagent uses `testthat::SummaryReporter`, which emits the failing `test_that()` description in each `Failure ('file:line:col'): <description>` header. Use that exact description verbatim as the `desc =` argument when you re-run the failing test inline.

## Triage & regression workflow

Use this loop when triaging failures or verifying a fix across a package. Skip it for single-file questions and for TDD on one new test.

1. **Run the full suite via the subagent** — its summary lists every failing file with the failing test names.
2. **Pick a failure, implement the fix.**
3. **Re-run only the failing test inline** with `Rscript -e 'devtools::load_all(); testthat::test_file("<path>", desc = "<exact description from the summary>")'`. Drop `desc =` only if the failure is order-dependent or the description is ambiguous.
4. **When green, re-run the file**, then re-run the full suite via the subagent to catch regressions elsewhere.
5. Repeat 2–4 until all pass.

If the user explicitly asks for the subagent on a single file, or for raw inline output on a multi-file run, honor the override.
