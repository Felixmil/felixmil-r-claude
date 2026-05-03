# felixmil-r-claude

Triage-focused R package testing for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Ships:

- **`/felixmil-r-claude:running-r-tests`** — a skill that picks the narrowest test scope ("run one test" vs "one file" vs "the whole suite"), drives the red-to-green workflow, and dispatches the subagent below for any multi-file run.
- **`r-test-runner`** — a context-isolating subagent that runs `testthat` and returns a condensed pass/fail/warn/skip summary plus extracted failure details. Verbose `Rscript` output stays out of the parent's context.
- **`/setup-r-test-runner`** — a one-time per-project skill that adds the Bash allow rules the subagent needs (`mkdir`, `printf`, `tee`, etc.) so it doesn't prompt on every dispatch. See [First-time setup](#first-time-setup-per-project).

## Why

Running a full test suite from inside Claude usually means the whole transcript fills with `testthat` output: every passing dot, every header, every diff. By the third file you're out of context. This plugin moves the noise into a subagent — the parent only sees the structured summary, with file paths, line numbers, and `test_that()` descriptions you can use as `desc =` filters for precise reruns.

## Install

In Claude Code:

```
/plugin marketplace add felixmil/felixmil-r-claude
/plugin install felixmil-r-claude@felixmil-r-claude
```

Then `/reload-plugins` (or restart Claude Code) to pick up the skill and agent.

## First-time setup (per project)

The subagent's setup steps (`mkdir` of the log dir, `printf` of `.gitignore`, `ln -sf` for the latest-log symlink, `tee` for log capture) need permission to run. Claude Code does not propagate plugin-shipped permission rules to subagent shells in consuming projects, so each project where you'll use the subagent needs the rules added once.

Two options:

**Option A — run `/setup-r-test-runner` once per project.** The skill writes the allow rules to `.claude/settings.local.json` (project-local, not committed) and confirms what it added.

**Option B — paste it yourself.** Add the following to `.claude/settings.local.json` at the package root:

```json
{
  "permissions": {
    "allow": [
      "Bash(Rscript -e 'testthat::test_file*)",
      "Bash(Rscript -e 'devtools::load_all*; testthat::test_file*)",
      "Bash(Rscript -e 'devtools::load_all*;testthat::test_file*)",
      "Bash(mkdir -p *)",
      "Bash(printf *)",
      "Bash(date *)",
      "Bash(ln -sf *)",
      "Bash(ls *)",
      "Bash(grep *)",
      "Bash(find *)",
      "Bash(rm -f *)",
      "Bash(tee *)"
    ]
  }
}
```

If you'd rather scope these globally instead of per-project, paste the same block into `~/.claude/settings.json`.

## Usage

Just talk about tests. The skill triggers on phrases like:

- "run the tests"
- "are tests passing?"
- "fix this failure"
- "rerun the failing test"

For multi-file runs the skill will dispatch the `r-test-runner` subagent. For single-file or single-test runs it'll run inline (faster, smaller output).

### Triage workflow

When you ask the parent to triage failures, the loop is:

1. Full suite via subagent → condensed summary listing failing tests by description.
2. Pick a failure, fix it.
3. Re-run that one test inline (`testthat::test_file(path, desc = "...")`) for fast feedback.
4. Once green, re-run the file, then re-run the full suite via the subagent again to catch regressions.

## Configure the log directory

The subagent writes logs and summaries under a configurable directory. **Default:** `./r-tests-runner/` (relative to the package root). The directory gets a self-ignoring `.gitignore` on first run, so it's safe to leave untracked.

To override, set `R_TEST_RUNNER_LOG_DIR` in your **user** or **project** `settings.json`'s `env` block (this plugin can't ship `env` defaults — only allow rules):

```json
{
  "env": {
    "R_TEST_RUNNER_LOG_DIR": "tmp/test-logs"
  }
}
```

The parent reads this in its own shell at dispatch time and passes the resolved path to the subagent — subagent shells may not inherit `env` from `settings.json`.

## What ships in this plugin

| File | Purpose |
| --- | --- |
| `agents/r-test-runner.md` | Subagent definition: fixed Bash templates, log persistence, output format. |
| `skills/running-r-tests/SKILL.md` | Skill definition: dispatch-by-scope rule + triage workflow. |
| `skills/setup-r-test-runner/SKILL.md` | One-time per-project setup: writes the subagent's Bash allow rules into `.claude/settings.local.json`. Invoke via `/setup-r-test-runner`. |
| `.claude/settings.json` | Permission rules for the parent session. Plugin-shipped permissions don't always reach subagent shells in consumer projects — see "First-time setup" above. |

## Requirements

- R with `testthat` (>= 3.0) and `devtools` installed.
- The R package under test must use testthat 3 (`Config/testthat/edition: 3` in `DESCRIPTION`).

## Companion skill

This plugin pairs well with [`r-lib:testing-r-packages`](https://github.com/posit-dev/skills) (Posit Dev skills marketplace) — that one tells Claude *how to write tests*; this plugin tells Claude *how to run and triage them*.

## License

[PolyForm Noncommercial 1.0.0](LICENSE.md). Free for personal, hobby, research, charitable, educational, and government use. Not licensed for commercial use.
