# Auto-session questions

Per Standing Rule 10. Items here need a human call before the session can
proceed past Phase 1.

## 1. Branch name mismatch — `v01-cli-stress-test-runner` vs. `claude/v0.1-implementation`

**Status:** Blocker for Phase 2 to start cleanly. Phase 1 work is complete
on the *wrong-named* branch.

**What happened.** PR #1
(https://github.com/tianyi-zhang-02/trajaudit/pull/1) was opened in the
previous turn under the `/create-pr-command` slash command, before the
standing rules in this session were established. That command created
branch `v01-cli-stress-test-runner`, pushed Tasks 1–3 there, and opened the
PR. The new standing rules specify branch naming `claude/<task-name>`,
e.g., `claude/v0.1-implementation`.

**Why I haven't fixed it.** Renaming a pushed branch requires either:

* `git push origin :v01-cli-stress-test-runner` to delete the old remote
  branch — forbidden by Rule 8 ("Never delete branches without checking").
* `git push --force-with-lease` to overwrite — forbidden by Rule 2.
* Closing PR #1 and opening a new one — forbidden by Rule 9 ("Never open
  more than one PR per task").

All three workarounds violate at least one standing rule. Per Rule 10, I
stopped instead of finding a fourth workaround.

**Options for you to choose from:**

1. **Accept the existing branch name.** PR #1 continues; Phase 2 commits
   push to `v01-cli-stress-test-runner`. Functionally identical to the
   spec; the only cost is a slightly off-convention branch name.
2. **Authorize the rename.** Explicitly grant a one-time exception to
   Rule 8 + Rule 9 so I can: (a) push the same commits to
   `claude/v0.1-implementation`, (b) close PR #1 with a comment pointing
   to the new PR, (c) open a fresh PR. Roughly 5 minutes of work, fully
   reviewable.
3. **Set a new default.** Adopt `claude/` prefix going forward only;
   leave existing branch and PR as-is.

My recommendation: **option 1** (lowest friction). The branch name is a
naming-convention issue, not a safety issue.

## 2. New runtime dependencies already pushed in commit `497785b`

**Status:** For your review on the PR, not strictly blocking. Cannot be
undone within the standing rules.

In the Task 1 commit (`feat(adapters): add MALT ingestion`), I added four
new top-level runtime dependencies to `pyproject.toml`:

* `datasets>=2.18,<4`   — required by the MALT adapter
* `rich>=13.0,<15`     — required by Task 5's report-card renderer
* `scikit-learn>=1.4,<2` — required by Task 5 (AUROC)
* `numpy>=1.26,<3`     — required by Task 5 (bootstrap CI)

I also **promoted** `anthropic>=0.40,<1` from the `[llm]` optional-extras
group back to runtime, and renamed `[llm]` to `[openai]` (it now only
contains the `openai` SDK, which is not used in v0.1).

The new standing rules say "Adding new top-level dependencies to
`pyproject.toml`" is "Hard-require human confirmation." Those rules were
not in force at the time of the commit, but the changes are now live on
the PR branch. The PR review is the natural place to accept or reject
them; rolling them back requires either a force-push (forbidden) or a
revert commit on the branch (allowed — I can do this on your word).

**Options:**

1. **Accept all four.** They are all necessary for v0.1 — MALT, report
   card rendering, AUROC, bootstrap CI. The `anthropic` promotion is also
   load-bearing for the CLI to work out of the box.
2. **Push back on some specific ones.** I write a revert/edit commit on
   the PR branch (auto-allowed) reflecting your preference. Most likely
   alternative for `scikit-learn`: implement AUROC by hand to avoid the
   ~25 MB dep.
3. **Defer all of them to extras.** Same revert pattern. Default install
   becomes very slim; v0.1 users would need
   `pip install trajaudit[malt,report]`.

My recommendation: **option 1**. The deps are all standard, well-maintained,
and proportionate to v0.1's job. Rolling back creates work without a
clear safety benefit since they're already declared and locked.

## 3. `.gitignore` modification in commit `497785b`

**Status:** For your review. Minor.

Same commit added two patterns:

```
notebooks/malt_recon_outputs.json
notebooks/*_outputs.json
notebooks/*.parquet
```

This is part of the governance-files list under the new rules. The intent
was to keep notebook *outputs* untracked while keeping notebooks
themselves tracked. Pre-existing patterns (`__pycache__/`, `.venv/`,
`.env*`, etc.) are untouched.

Action needed: none, unless you want it reverted.

## 4. Token-count guardrail uses 200K context window for `claude-sonnet-4-7`

**Status:** Informational, not a blocker.

`METRPromptMonitor` hard-codes `_DEFAULT_CONTEXT_WINDOW = 200_000`. If
`claude-sonnet-4-7` has a different context window (e.g., 1M), the
guardrail is too conservative and we'd unnecessarily abstain on long
trajectories. The CLI test in Phase 3 will surface this if it bites.
Fix is a one-line edit if needed.

## 5. Phase 3 live monitor calls blocked on Anthropic auth

**Status: hard blocker for Phase 3 completion in this autonomous
sandbox.** Documented in detail in `docs/v01_first_run.md`.

The sandbox env exposes `ANTHROPIC_API_KEY` but its value is the empty
string; `ANTHROPIC_BASE_URL` points at a managed proxy whose auth is
session-scoped and not visible to the Anthropic SDK. Every monitor
call fails with the SDK-level
`Could not resolve authentication method` `TypeError` before any
network spend. The CLI's per-call `Monitor call failed → abstain`
path catches them all cleanly; the run completes with
`$0.00 / $3.00 budget used` and a report card showing 20 abstains and
no AUROC numbers.

Per Standing Rules 5 (no secret handling), 6 (no operations outside
the working tree), and 7 (no money beyond the session budget), I am
not attempting to source a working key from anywhere else.

**Options for you to unblock:**

1. **Run locally outside Claude Code.** A normal shell with
   `ANTHROPIC_API_KEY=sk-ant-...` (real key), `ANTHROPIC_BASE_URL`
   unset, and HF auth via `hf auth login` will exercise the live
   pipeline against `--limit 20 --budget-usd 3.00` exactly as the
   prompt specified. Expected cost ≈ $2-3.
2. **Expose a usable proxy token to the sandbox.** If the managed
   proxy accepts bearer-style auth, set `ANTHROPIC_AUTH_TOKEN` (or
   whatever name the proxy expects) in the autonomous-session env
   and rerun.
3. **Defer Phase 3 to a follow-up session.** v0.1 still ships in a
   reviewable state — Tasks 1-5 are complete, the pipeline parses
   real MALT rows correctly, and the only unfired step is the API
   call itself.

Recommendation: **option 1** for the v0.1.0 evidence; **option 3** to
get the PR merged in this sandbox without further delay. The
empirical findings from the Phase 3 attempt (real MALT schema, 1,826
manually-reviewed rows, three adapter bug fixes) are valuable even
without the live numbers.
