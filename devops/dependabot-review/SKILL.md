---
name: dependabot-review
description: >
  Weekly Dependabot PR triage and merge cycle. Lists all open Dependabot PRs,
  evaluates each for risk, merges safe ones, and reports on any that need
  manual attention. Run with /dependabot-review on Sundays.
tools: Bash, Read, Glob, Grep
---

# Dependabot PR Review Cycle

Triage and merge open Dependabot PRs. Follow every phase in order.

---

## Phase 1 — List open Dependabot PRs

```bash
gh pr list --author "app/dependabot" --state open --json number,title,labels,createdAt,url --jq '.[] | "\(.number)\t\(.title)\t\(.url)"'
```

If there are no open PRs: state "No open Dependabot PRs — nothing to do." and stop.

Otherwise, list them in a table for the user.

---

## Phase 2 — Triage each PR

For each open Dependabot PR, gather information:

1. **Check CI status**:

   ```bash
   gh pr checks <number>
   ```

2. **Read the PR diff** to understand the scope of the change:

   ```bash
   gh pr diff <number> --name-only
   gh pr view <number> --json body --jq .body
   ```

3. **Categorise the PR**:

   | Category       | Criteria                                                        | Action        |
   | -------------- | --------------------------------------------------------------- | ------------- |
   | **Auto-merge** | Patch/minor update, CI passes, no breaking changes in changelog | Merge         |
   | **Review**     | Major update, CI passes, changelog mentions breaking changes    | Flag for user |
   | **Failing CI** | Any update where CI is failing                                  | Flag for user |
   | **Stale**      | PR open > 30 days with no activity                              | Flag for user |

4. Present a summary table to the user:

   | PR  | Title | Category | CI  | Recommendation |
   | --- | ----- | -------- | --- | -------------- |

---

## Phase 3 — Merge safe PRs

For each PR categorised as **Auto-merge**:

1. Approve the PR:

   ```bash
   gh pr review <number> --approve --body "Dependabot weekly triage: patch/minor update, CI green. Auto-merging."
   ```

2. Merge with squash:

   ```bash
   gh pr merge <number> --squash --auto --delete-branch
   ```

3. State: "Merged PR #<number>: <title>"

If `--auto` is not enabled on the repo, merge directly:

```bash
gh pr merge <number> --squash --delete-branch
```

---

## Phase 4 — Report

After processing all PRs, present a final summary:

```
## Dependabot Weekly Summary — <date>

### Merged
- PR #<n>: <title>

### Needs Manual Review
- PR #<n>: <title> — Reason: <major update / failing CI / stale>

### Statistics
- Total open: <n>
- Auto-merged: <n>
- Flagged for review: <n>
```

---

## Phase 5 — Update security alerts

Check if any Dependabot security alerts remain after merging:

```bash
gh api repos/{owner}/{repo}/dependabot/alerts --jq '[.[] | select(.state == "open")] | length'
```

If alerts remain, list them:

```bash
gh api repos/{owner}/{repo}/dependabot/alerts --jq '.[] | select(.state == "open") | "\(.security_advisory.severity)\t\(.security_advisory.summary)\t\(.html_url)"'
```

State how many open alerts remain and their severity.

---

## Execution rules

- Never force-merge a PR with failing CI.
- Never merge a major version bump without user confirmation.
- If a PR has merge conflicts, flag it for the user — do not attempt to resolve.
- If multiple Dependabot PRs update the same package (e.g., grouped vs individual), prefer the grouped PR.
- After merging, wait 10 seconds before merging the next PR to avoid GitHub rate limits.
