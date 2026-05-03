---
name: setup-r-test-runner
description: One-time per-project setup for the felixmil-r-claude plugin. Adds the Bash allow rules the `r-test-runner` subagent needs (mkdir, printf, ln, tee, etc.) to the project's `.claude/settings.local.json` so the subagent can run without permission prompts. Use when the user says "set up r-test-runner", "configure r-test-runner permissions", "the r-test-runner subagent keeps asking for permission", "set up the test runner for this project", or runs `/setup-r-test-runner`.
---

# Setup r-test-runner permissions for this project

The `r-test-runner` subagent that ships with this plugin issues a fixed set of Bash commands during setup (`mkdir`, `printf`, `date`, `ln -sf`, `ls`, `grep`, `find`, `rm -f`, `tee`) plus templated `Rscript -e 'testthat::test_file(...)'` invocations. Claude Code does not propagate plugin-shipped allow rules to subagent shells in consuming projects, so each project where the subagent will be used needs these rules added once.

This skill writes those rules to **`.claude/settings.local.json`** at the project root — local-only, not committed by default.

## What the skill does

1. Resolve the project root. Use the cwd. If `cwd` is not the package root (no `DESCRIPTION` next to it), tell the user and stop — don't guess.
2. Read `.claude/settings.local.json` if it exists; create the directory and an empty `{}` file if it doesn't. Use the Read tool, then the Write or Edit tool. Do not shell out for this.
3. Merge the rule set below into `permissions.allow`. Do not duplicate rules already present. Preserve every other key in the file unchanged. Sort the final list lexicographically for stability.
4. Show the user the diff (added rules only) and the final file path. Don't be chatty.

## The rule set to merge

```
Bash(Rscript -e 'testthat::test_file*)
Bash(Rscript -e 'devtools::load_all*; testthat::test_file*)
Bash(Rscript -e 'devtools::load_all*;testthat::test_file*)
Bash(mkdir -p *)
Bash(printf *)
Bash(date *)
Bash(ln -sf *)
Bash(ls *)
Bash(grep *)
Bash(find *)
Bash(rm -f *)
Bash(tee *)
```

These match exactly what the subagent's templates issue — no broader. The `Rscript` rules cover the inline `testthat::test_file(...)` / `devtools::load_all(); testthat::test_file(...)` patterns the parent skill uses for narrow reruns, so single-test reruns also stop prompting.

## After writing

Tell the user:

- The rules were added to `<project-root>/.claude/settings.local.json`.
- They take effect on the next user message (no restart needed in Claude Code; the harness re-reads `settings.local.json` between turns).
- If they prefer the rules globally instead, the same block can be moved into `~/.claude/settings.json` (mention only if they ask).

## Edge cases

- **File already has every rule:** report "Already configured — nothing to do." Don't rewrite the file.
- **File has a malformed JSON:** don't overwrite. Show the parse error, point at the file, and stop. Let the user fix it.
- **`permissions.allow` exists but is not a JSON array:** don't coerce. Stop and report.
- **No `DESCRIPTION` at cwd:** the subagent expects to run from the package root. Stop and ask the user to invoke the skill from the package root, or to confirm the path.

## What this skill does NOT do

- Touch `.claude/settings.json` (the committed/shared one). Project-local opt-in is the right scope here — committing the allow list assumes every contributor wants it.
- Touch `~/.claude/settings.json`. Global scope is the user's call.
- Add `R_TEST_RUNNER_LOG_DIR`. That's an env var, not a permission, and the README covers it separately.
- Verify R or `testthat` is installed. The subagent surfaces those errors clearly when it actually runs.
