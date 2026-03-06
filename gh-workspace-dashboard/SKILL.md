---
name: gh-workspace-dashboard
description: Cross-repo GitHub workspace dashboard using gh. Lists and prioritizes your open PRs across configured repos (CI status, review state, who is blocking). Use when the user asks "What do I need to code today?", "Show my open PRs and what's blocking", or wants a summary of PRs and next actions across multiple repos.
---

# GH workspace dashboard

Orchestrates **cross-repo** PR visibility and next actions using `gh`. Use this when the user asks for a coding-priority overview across their key repos. Requires `gh` to be installed and authenticated (`gh auth status`).

## Scope (edit to match your repos and users)

- **Repos**: AMDResearch/intellikit, ROCm/iris, AMDResearch/kerneldb. Add or remove repos in the list below; these are the repos to scan for \"your\" PRs.
- **Authors**: PRs authored by **mawad-amd** and Copilot-related bots (for example `app/copilot-swe-agent`, `github-copilot[bot]`, `copilot-pull-request-reviewer`). Adjust this list to match the actual GitHub logins used in your org.

## Entry prompts

- "What do I need to code today?"
- "Show me my open PRs and what's blocking."
- "What's blocking across my PRs?"

When the user asks in this spirit, run the workflow below and summarize results in the priority order given.

## Workflow

### 1. List open PRs across repos

For each repo in scope, list open PRs by **you and Copilot** (using the authors from the Scope section):

```bash
for repo in AMDResearch/intellikit ROCm/iris AMDResearch/kerneldb; do
  for author in mawad-amd app/copilot-swe-agent github-copilot[bot] copilot-pull-request-reviewer; do
    gh pr list -R "$repo" --author "$author" --state open \
      --json number,title,headRefName,url,reviewRequests,repository
  done
done
```

Adjust the `author` list to match the actual GitHub logins used in your environment. Merge or concatenate the outputs so you have one list of all open PRs.

### 2. Enrich each PR: CI + review state

For each PR, get:

- **CI / status checks**: `gh pr view <number> -R <owner/repo> --json statusCheckRollup` or `gh pr checks <number> -R <owner/repo>`
- **Review requests**: already in `reviewRequests` from step 1
- **Review state**: `gh pr view <number> -R <owner/repo> --json reviews,reviewDecision` if needed

Use this to classify each PR (see prioritization below).

### 3. Prioritize and summarize

Order and present PRs in this priority:

1. **Red CI on your branches** — CI failing; needs your fix before merge.
2. **PRs waiting on you** — Reviews requested from you, or comments you haven’t answered; you are the blocker.
3. **PRs waiting on others** — Review requested from others and/or CI green; you’re unblocked, others need to act.

For each PR, show: repo, number, title, branch, CI status (green/red/pending), and who it’s waiting on (you vs others). Give a one-line “next action” where useful (e.g. “Fix lint”, “Address review comments”, “Ready for merge once reviewed”).

## Optional rules (when to act)

- **When to `gh pr comment`** — e.g. when the user says “tell Copilot to fix lint on this PR”: use `gh pr comment <number> -R <owner/repo> --body "@copilot use gh client and fix lint"` (or the agreed wording). Only do this when the user explicitly asks to comment.
- **When to `gh issue create`** — When the user asks to open an issue and assign to a teammate, use `gh issue create -R <owner/repo> --title "..." --body "..." --assignee <login>`. Confirm repo and assignee before creating.
- **When to ping people** — Only when the user explicitly asks to ping or re-request reviewers; use `gh pr edit <number> -R <owner/repo> --add-reviewer <login>` or a comment @mention as requested.

## Summary

1. Resolve scope (repos + author) from this file.
2. List open PRs by author for each repo.
3. For each PR, fetch statusCheckRollup and review state.
4. Sort by: red CI → waiting on you → waiting on others.
5. Return a short dashboard and next actions; run `gh pr comment` / `gh issue create` only when the user asks.

This skill is the “chain of command” starting point for “what do I need to code today?” across the configured repos.
