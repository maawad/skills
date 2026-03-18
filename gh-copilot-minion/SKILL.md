---
name: gh-copilot-minion
description: Uses GitHub CLI (gh) to delegate trivial or mechanical fixes on a PR to the Copilot SWE agent via a comment, then reviews and merges Copilot’s sub-PR and lets the main PR benefit. Use when a PR is blocked on fixable CI/lint or small cleanups and the user wants Copilot to handle it end-to-end.
---

# GitHub Copilot “minion” flow

This skill encodes the Copilot delegation pattern:

You → Agent → `gh` + comment → **Copilot sub-PR** → Agent reviews/merges → main PR CI recovers.

Requires `gh` to be installed and authenticated (`gh auth status`), and the Copilot SWE agent (`app/copilot-swe-agent`) enabled in the repo.

## When to use

- A main PR (for example `ROCm/iris#391`) is **blocked on CI** (lint/format, tests, or other fixable failures), or needs a **non-trivial refactor / larger change** you’re comfortable delegating.
- The user explicitly wants to **delegate the work to Copilot**, not hand-edit code.
- The repo already uses the Copilot SWE agent and you expect it to open follow-up PRs on your branch.

For **large refactors or security-sensitive changes**, it’s fine to delegate to Copilot, but:

- Expect to spend more time on diff review and testing before merge.
- Do not auto-merge; keep the “Agent reviews/merges” step strict and conservative.

## Workflow

### 1. Detect the upstream problem on the main PR

Given a main PR number `<main-num>` and repo `<owner>/<name>` (for example `ROCm/iris`):

```bash
gh pr checks <main-num> -R <owner>/<name>
```

Look for:

- Failing **lint/format** jobs.
- Other clearly mechanical CI failures Copilot can likely fix (e.g. missing imports, style violations).

Optionally re-run failed jobs first if they look flaky:

```bash
gh run list -R <owner>/<name> --limit 10
gh run rerun <run-id> --failed -R <owner>/<name>
```

If failures are real and look fixable by Copilot, continue.

### 2. Delegate the fix to Copilot via comment

When addressing the Copilot SWE agent on a PR or issue you **must use @copilot** (not @app/copilot-swe-agent) at the start of the comment so it gets notified and acts on the request.

> **Assigning Copilot:** To assign Copilot to an issue, use the username `copilot-swe-agent` (no `[bot]` suffix). Variants like `Copilot`, `copilot`, or `copilot-swe-agent[bot]` will **not** work with `gh issue edit --add-assignee`. Example:
> ```bash
> gh issue edit <issue-num> -R <owner>/<name> --add-assignee "copilot-swe-agent"
> ```

On the main PR, post a comment asking Copilot to use `gh` and fix failures, for example:

```bash
gh pr comment <main-num> -R <owner>/<name> \
  --body "@copilot use gh client and see linter errors and fix"
```

Copilot should respond by opening a **sub-PR** on the same branch (for example `ROCm/iris#430`) authored by `app/copilot-swe-agent`. Watch for new PRs with the same `headRefName` as the main PR.

### 3. Review Copilot’s sub-PR

Once the Copilot PR exists (call it `<sub-num>`):

```bash
gh pr view <sub-num> -R <owner>/<name> --json number,title,headRefName,url
gh pr diff <sub-num> -R <owner>/<name>
gh pr checks <sub-num> -R <owner>/<name>
```

Verify:

- The diff only contains the expected changes (for example, ruff/formatting, small obvious code fixes).
- CI on the sub-PR is **green** or at least not introducing new obvious failures.

If anything looks off, stop and ask the user before merging.

### 4. Undraft (if needed) and merge the sub-PR

If Copilot opened the PR as a draft:

```bash
gh pr ready <sub-num> -R <owner>/<name>
```

Then merge normally (no force push):

```bash
gh pr merge <sub-num> -R <owner>/<name> --merge
```

If GitHub is slow to move the PR out of draft, retry the merge after a short delay.

### 5. Let the main PR benefit from the fixes

Merging the Copilot sub-PR updates the branch behind the main PR. After merge:

```bash
gh pr checks <main-num> -R <owner>/<name>
```

If CI did not auto-rerun or is still flakey:

```bash
gh run list -R <owner>/<name> --limit 10
gh run rerun <run-id> --failed -R <owner>/<name>
```

Confirm that:

- Lint/format jobs now pass.
- The main PR is no longer blocked on the issues Copilot fixed.

### 6. View Copilot session logs

You can inspect what Copilot is doing (or did) in real time using the `gh agent-task` CLI or the Copilot sessions API.

**List recent sessions:**

```bash
gh agent-task list -L 10
```

**View a session (interactive only — requires session ID):**

```bash
gh agent-task view <session-id> --log
```

**Get session IDs and logs via the API (works non-interactively):**

The `gh agent-task` CLI hits `https://api.githubcopilot.com/agents/sessions` under the hood. Use `curl` to access it directly:

```bash
TOKEN=$(gh auth token)

# List sessions (most recent first)
curl -s -H “Authorization: Bearer $TOKEN” -H “Accept: application/json” \
  “https://api.githubcopilot.com/agents/sessions?page_number=1&page_size=5&sort=last_updated_at,desc”
```

Each session object includes: `id`, `name`, `state` (`in_progress`/`completed`/`failed`), `resource_number` (PR number), `repo_id`, `model`, `workflow_run_id`, `created_at`, `completed_at`.

**Stream live session logs:**

```bash
# Get the session ID from the list above, then fetch logs:
curl -s -H “Authorization: Bearer $TOKEN” -H “Accept: application/json” \
  “https://api.githubcopilot.com/agents/sessions/<session-id>/logs”
```

The logs endpoint returns Server-Sent Events (SSE) with the agent's reasoning, tool calls, and command outputs. Extract readable content with:

```bash
curl -s -H “Authorization: Bearer $TOKEN” -H “Accept: application/json” \
  “https://api.githubcopilot.com/agents/sessions/<session-id>/logs” \
  | grep -o '”content”:”[^”]*”' | sed 's/”content”:”//;s/”$//' | grep -v “^null$”
```

> **Note:** The Copilot agent runs on self-hosted runners (same as CI). It has access to GPUs and can execute commands directly — it does not need a CI workflow to run benchmarks or tests.

**Cancel a running session:**

If Copilot is stuck (e.g. a hung command that will never return), cancel the session via `PUT`:

```bash
TOKEN=$(gh auth token)
curl -X PUT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"state":"cancelled"}' \
  "https://api.githubcopilot.com/agents/sessions/<session-id>"
```

> **Interacting with a running session:** There is no API to send messages into a live session. You can only **cancel** it (via `PUT` above) and then **post a new PR comment** mentioning `@copilot` to start a fresh session with updated instructions. The `gh agent-task` CLI also has no "message" or "reply" subcommand — only `list`, `view`, and `create`.

## Summary

1. Use `gh pr checks` on the main PR to confirm it is blocked on fixable CI/lint issues.
2. Post a targeted `@copilot` comment asking it to inspect CI/lint and fix.
3. When Copilot opens a sub-PR, review its diff and CI.
4. Undraft (if needed) and merge the sub-PR with a normal merge.
5. Re-check CI on the main PR and rerun failed jobs if necessary.
6. Use `gh agent-task list` or the `api.githubcopilot.com` sessions API to monitor Copilot's progress and inspect its reasoning logs.

Use this pattern whenever you want Copilot to act as a “minion” that cleans up CI/lint and small fixes on your branches, while you stay in control of review and merge.

