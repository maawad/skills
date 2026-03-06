## maawad/skills

Personal AI skills for dev workflows (Cursor, Codex, Claude, etc.).

### Skills

- **pre-push-hygiene**: Pre-push checklist (format, tests, commit messages, secrets).
- **gh-pr-review**: `gh`-based PR review, threads, and re-requesting reviewers.
- **gh-workspace-dashboard**: Cross-repo “what do I need to code today?” — list and prioritize your open PRs (CI, reviews, who’s blocking) across configured repos.
- **gh-copilot-minion**: Copilot SWE agent “minion” flow to fix CI/lint or small issues via sub-PRs, then review and merge them.
- **read-the-damn-code**: Clone upstream (Triton, ROCm) and read source instead of guessing.

### Install (Cursor)

Project-local (into `.cursor/skills`):

```bash
curl -sSL https://raw.githubusercontent.com/maawad/skills/main/install/skills/install.sh \
  | bash -s -- --target cursor
```

Global (into `~/.cursor/skills`):

```bash
curl -sSL https://raw.githubusercontent.com/maawad/skills/main/install/skills/install.sh \
  | bash -s -- --target cursor --global
```

