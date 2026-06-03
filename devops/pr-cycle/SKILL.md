---
name: pr-cycle
description: >
  Autonomous PR cycle: run the full branch→test→lint→fix→commit→push→PR loop
  without manual intervention. Self-corrects lint and test failures before
  opening a PR. Use for any code change ready to ship.
tools: Bash, Read, Glob, Grep, Edit, Write
---

# PR Cycle

Run the complete branch → implement → test → lint → fix → PR cycle autonomously.
Do not ask for input unless there is an external blocker (permissions error, missing secret, etc.).

---

## Step 1 — Verify branch

```bash
git branch --show-current
```

**Never run this skill on `main`.** If on main, stop and ask the user which branch to use.

---

## Step 2 — Test + lint loop

Run all suites in order. If anything fails, fix it and restart the loop from the top. Keep looping until everything is green.

### 2a — TypeScript

```bash
cd /c/Users/srely/Repos/portfolio
npm run check
```

### 2b — Unit tests

```bash
npm run test:unit
```

### 2c — API tests

```bash
python -m pytest tests/ -v
```

### 2d — Python lint

```bash
flake8 api/ --max-line-length=120
```

### 2e — Markdown lint

```bash
npx markdownlint-cli2 'docs/**/*.md'
```

**Fixing failures:**

- TypeScript errors → fix the type mismatch in the source file; do not add `// @ts-ignore`
- Test failures → fix the implementation, not the test (unless the test itself is wrong)
- flake8 → fix import order (E401), line length (E501), unused imports (F401)
- markdownlint → MD025 = only one H1 per file; MD041 = first line must be H1

---

## Step 3 — Sync memory → planning board

Run the `sync-memory-to-clineboard` skill to update `roadmap.md` and `wip.md` before the docs commit.
This keeps the Kanban and Recently Completed table in sync with MEMORY.md.

Stage the planning docs together with any other docs changes — they go in the same commit.

---

## Step 3b — Audit docs nav coverage

After any PR that adds, removes, or renames `.md` files under `docs/`, verify nav coverage:

```bash
# List all .md files on disk
find docs -name "*.md" | grep -v "__pycache__" | sort

# List all files referenced in nav
grep -r "\.md" docs/.nav.yml docs/**/.nav.yml | grep -v "^Binary" | sort
```

Cross-reference both lists. For every `.md` file NOT in the nav, decide:

| Case                                                        | Action                                                                            |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Active doc (feature plan, reference, runbook)               | Add to the nearest section's `.nav.yml` AND to `docs/.nav.yml`                    |
| Archived / handover / one-off snapshot                      | Move to `docs/planning/archived/` and add under a Planning → Archived sub-section |
| Generated file (cost-dashboard auto-committed, coverage.md) | Already wired — verify it is still present in nav after changes                   |

Update **both** the root `docs/.nav.yml` and the relevant sub-directory `.nav.yml` when adding entries — they must stay in sync.

Stage nav file changes in the same commit as the content changes that triggered them.

---

## Step 4 — Commit

Stage only relevant files (never `git add -A` blindly):

```bash
git add <specific files>
git status
```

Commit using conventional commit format:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <short summary>

<optional body>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

**Conventional commit types:**

| Type       | When to use                           |
| ---------- | ------------------------------------- |
| `feat`     | New feature                           |
| `fix`      | Bug fix                               |
| `docs`     | Documentation only                    |
| `refactor` | Code restructure, no behaviour change |
| `test`     | Adding or fixing tests                |
| `chore`    | Build, deps, CI config                |
| `style`    | Formatting, lint fixes                |

---

## Step 5 — Push

```bash
git push -u origin $(git branch --show-current)
```

---

## Step 6 — Open PR

```bash
gh pr create \
  --title "<type>(<scope>): <summary>" \
  --body "$(cat <<'EOF'
## Summary
- <bullet 1>
- <bullet 2>

## Test plan
- [ ] Unit tests pass
- [ ] API tests pass
- [ ] Lint clean

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Report the PR URL when done.

---

## Quick reference — which suites to run per change

| Changed area             | Minimum suites            |
| ------------------------ | ------------------------- |
| React components / hooks | TypeScript + unit + E2E   |
| FastAPI / Python backend | API tests + lint          |
| Bicep / infra / YAML     | None locally — rely on CI |
| Docs / markdown          | Markdownlint              |
| Styles / layout          | E2E                       |

---

## Hard rule — always test locally before pushing

**Run the relevant suites from Step 2 before every `git push`.** Never push code
that has not been locally verified. CI failures that could have been caught locally
waste review cycles and break the PR.

Specifically:

- When touching `api/requirements.txt`: verify the package exists on PyPI with
  `pip index versions <package>` before adding a pinned version.
- When migrating a component from a local fetch/state pattern to a shared context
  (e.g. `FeatureFlagsContext`): update **all** tests that mock the old fetch endpoint
  to mock the context instead — then run `npm run test:unit` to confirm before pushing.
- When adding a new Python dependency: run `pip install -r api/requirements.txt` in
  a clean venv (or at minimum verify the package name resolves) before committing.
