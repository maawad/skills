---
name: gh-pr-review
description: Use GitHub CLI (gh) for pull request workflows: inspect PR status and checks, list and resolve review threads (GraphQL), re-request reviewers, view runs. Use when the user asks to address PR comments, resolve review threads, re-request Copilot or other review, or check CI/PR state.
---

# GitHub CLI – PR and review workflows

Use `gh` (and `gh api graphql` where needed) for PR state, review threads, and reviewers. Requires `gh` to be installed and authenticated (`gh auth status`).

## PR status and view

- **Current branch PR**: `gh pr status`
- **PR details (JSON)**: `gh pr view <number> --json number,title,headRefName,reviewRequests,reviews`
- **Checks**: `gh pr checks [number]` or `gh run list` in repo

## Review threads (comments on code)

Threads are not in `gh pr view`; use GraphQL.

**List threads and resolution state** (replace `$owner`, `$name`, `$number`):

```bash
gh api graphql -f query='
query($owner:String!,$name:String!,$number:Int!){
  repository(owner:$owner,name:$name){
    pullRequest(number:$number){
      number title
      reviewThreads(first:50){
        nodes{ id isResolved isOutdated path line
          comments(first:5){ nodes{ author{login} body createdAt } }
        }
      }
    }
  }
}' -f owner=OWNER -f name=REPO -F number=PR_NUMBER
```

**Resolve a thread** (after addressing the comment):

```bash
gh api graphql -f query='
mutation($id:ID!){
  resolveReviewThread(input:{threadId:$id}){
    thread{ id isResolved }
  }
}' -f id=PRRT_xxxx
```

Use the `id` from `reviewThreads.nodes[].id` (e.g. `PRRT_kwDOOGlCeM5yfyne`).

## Re-request a reviewer

```bash
gh pr edit <number> --add-reviewer <login>
```

Example: `gh pr edit 70 --add-reviewer copilot-pull-request-reviewer`

## Repo identity
For GraphQL, get owner/name: `gh repo view --json nameWithOwner`.

## Typical workflow

1. `gh pr status` to get current PR number.
2. List threads with the query above; find unresolved ones and their `id`.
3. After addressing comments in code, resolve each with the mutation.
4. Optionally re-request review: `gh pr edit <number> --add-reviewer <login>`.
