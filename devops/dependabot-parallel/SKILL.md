---
name: dependabot-parallel
description: >
  Parallel Dependabot PR triage. Spawns one sub-agent per open Dependabot PR
  to check CI, detect conflicts, merge clean PRs, rebase conflicted ones, and
  flag failures. Use when 5+ Dependabot PRs are open. For smaller batches,
  use /dependabot-review instead.
tools: Bash, Agent
---

# Dependabot Parallel Triage

Triage all open Dependabot PRs simultaneously using parallel sub-agents.
Each agent handles one PR independently — no sequential waiting.

---

## Step 1 — List open Dependabot PRs

```bash
gh pr list --author app/dependabot --json number,title,headRefName,mergeable,statusCheckRollup \
  --jq '.[] | "\(.number) \(.title)"'
```

If there are 0 open PRs, stop and report "No open Dependabot PRs."

If there are fewer than 5, consider using `/dependabot-review` instead (simpler, sequential).

---

## Step 2 — Spawn one sub-agent per PR

Use the Agent tool to launch all agents in a single message (parallel).

For each PR number N, spawn an agent with this prompt:

```
Triage Dependabot PR #N in the SvenRelijveld1995/portfolio repo.

1. Run: gh pr checks N --watch
   - If all checks pass → proceed to merge
   - If any check is failing → skip merge, add a comment and flag it

2. Run: gh pr view N --json mergeable --jq '.mergeable'
   - If CONFLICTING → attempt rebase:
       gh pr checkout N
       git rebase main
       git push --force-with-lease
       gh pr checks N --watch   (wait for re-run)
   - If MERGEABLE → proceed to merge

3. Merge (only if CI green and no conflicts):
   gh pr merge N --squash --auto --delete-branch

4. Report back: PR #N — <merged|rebased-and-merged|flagged: reason>
```

---

## Step 3 — Collect results

Wait for all agents to finish, then output a summary table:

```
| PR # | Title | Disposition |
|------|-------|-------------|
| 123  | bump lodash from 4.17.20 to 4.17.21 | merged |
| 124  | bump typescript from 5.0.0 to 5.1.0  | flagged: CI failing (test-frontend) |
| 125  | bump fastapi from 0.100.0 to 0.101.0  | rebased-and-merged |
```

---

## Step 4 — Security alert check

After triage, check for any Dependabot security alerts that weren't covered by open PRs:

```bash
gh api repos/SvenRelijveld1995/portfolio/vulnerability-alerts 2>/dev/null \
  && echo "Vulnerability alerts present — review manually" \
  || echo "No vulnerability alerts"
```

---

## Guard-rails

- **Never merge a PR with failing CI** — flag it instead
- **Never force-merge** — always use `--auto` (waits for CI)
- **Never silently skip a major version bump** — flag for manual review with the note "major version bump"
- **Never rebase more than once** — if the second rebase still conflicts, flag it
