---
name: pre-push-hygiene
description: Enforces pre-push checks: code formatting (ruff, clang-format), unit tests, commit message format ("Did x"), and secret scanning. Use when setting up or running pre-push hooks, validating before push, or when the user asks about push hygiene, commit message format, or preventing secrets in commits.
---

# Pre-push hygiene

Before pushing, run these checks (or help the user run them). Use the checklist and commands below.

## Respect project guides and scripts

- If the repo has a contributing guide (e.g. `CONTRIBUTING.md`, `docs/contributing.md`), follow its instructions for formatting, tests, and commits **before** using the defaults in this skill. Those project-specific rules take precedence.
- Before inventing commands yourself, look for existing scripts and targets (e.g. `Makefile`, `scripts/`, `package.json` scripts, `tox.ini`, `noxfile.py`, `pre-commit` config, CI workflows) and prefer running those. Use the commands below as fallbacks when no project guidance exists.


## Checklist

- [ ] **Formatting** — Python (ruff), C/C++ (clang-format), and any project-specific formatters pass
- [ ] **Unit tests** — Test suite passes
- [ ] **Commit messages** — All commits being pushed follow the "Did x" format
- [ ] **Secrets** — No API keys, tokens, or credentials in staged/committed files

---

## 1. Formatting

Run any existing project format commands first if they exist (for example, those documented in the contributing guide, `Makefile` targets, `scripts/`, `pre-commit` configs, or CI workflows). Use the commands below only when the project does not already define a formatter entry point.

**Python (ruff)**

```bash
ruff check . && ruff format --check .
```

To fix and then re-check:

```bash
ruff check --fix . && ruff format .
```

**C/C++ (clang-format)**

Check only (no write):

```bash
find . -type f \( -name '*.c' -o -name '*.cpp' -o -name '*.h' -o -name '*.hpp' \) ! -path './build/*' ! -path './.git/*' -exec clang-format --dry-run --Werror {} +
```

Fix in place:

```bash
find . -type f \( -name '*.c' -o -name '*.cpp' -o -name '*.h' -o -name '*.hpp' \) ! -path './build/*' ! -path './.git/*' -exec clang-format -i {} +
```

**Other formatters** — If the project uses Prettier, gofmt, black (legacy), etc., run the project’s usual format/check command (e.g. `pre-commit run --all-files` if they use pre-commit).

---

## 2. Unit tests

Run the project’s test runner. Prefer the same command the CI uses, and any commands documented in the repo’s contributing guide or scripts (e.g. `CONTRIBUTING.md`, `Makefile`, `scripts/`, `package.json`, `tox.ini`, CI workflows). Use the commands below only when no project-specific guidance exists.

**Python (pytest)**

```bash
pytest
# or: python -m pytest
```

**C/C++ (CTest)**

```bash
cd build && ctest
```

**Other** — Use the repo’s documented test command (e.g. `make test`, `cargo test`, `go test ./...`).

---

## 3. Commit message format

Commits being pushed must use the **"Did x"** style: imperative, past-tense summary of what was done.

**Good examples**

- `Implemented a Virtual Memory 0-based allocator`
- `Fixed null pointer dereference in parser`
- `Added unit tests for the allocator`

**Bad examples**

- `WIP` / `fix` / `updates`
- `Implemented a Virtual Memory 0-based allocator.` (trailing period optional but keep consistent)
- `implemented...` (use sentence case: "Implemented")

**Check commits being pushed**

```bash
git log --oneline origin/main..HEAD
# or, if pushing to current branch’s upstream:
git log --oneline @{u}..HEAD
```

Review each line; if any message doesn’t follow "Did x", suggest a rewrite and have the user amend:

```bash
git commit --amend -m "Implemented a Virtual Memory 0-based allocator"
```

---

## 4. Secret scanning

Ensure no secrets (API keys, tokens, passwords, private keys) are in the commits being pushed.

**Option A — gitleaks (recommended)**

```bash
gitleaks detect --no-git --source . -v
# Or against the diff being pushed:
gitleaks detect --log-opts="-p @{u}..HEAD" -v
```

**Option B — Quick grep over staged + committed**

Scan for common patterns in files that are staged or in the push range (exclude binaries and vendored deps):

```bash
git diff --cached --name-only && git diff --name-only @{u}..HEAD
```

Then search for high-risk patterns (adjust paths as needed):

- `api[_-]?key\s*=\s*['\"]?[a-zA-Z0-9_\-]{20,}`
- `secret\s*=\s*['\"]?[a-zA-Z0-9_\-]{20,}`
- `password\s*=\s*['\"]?\S+`
- `Bearer\s+[a-zA-Z0-9_\-\.]+`
- `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`
- `.env` files with secrets (ensure `.env` is in `.gitignore` and never committed)

If anything suspicious appears, treat it as a potential secret: remove from history (e.g. `git filter-branch` / `git filter-repo` or BFG), rotate the credential, and fix the commit.

---

## Workflow summary

1. Run formatters (ruff, clang-format, or project default); fix any failures.
2. Run the test suite; fix any failures.
3. Inspect `git log @{u}..HEAD` and fix any commit messages that don’t follow "Did x".
4. Run secret detection (gitleaks or grep); resolve any findings before pushing.
5. Push only when all four checks pass.

