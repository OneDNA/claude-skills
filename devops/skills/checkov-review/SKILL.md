---
name: checkov-review
description: >
  Fetch the latest validate-infrastructure run, parse checkov results, and
  write docs/planning/checkov-resolve-plan.md with a triage of failed checks:
  which to fix in Bicep, which to skip (with justification), and any parse errors.
tools: Bash, Read, Write
---

# Checkov Review

Fetch the latest Checkov output from the `validate-infrastructure` workflow,
triage each finding, and write an actionable resolve plan to
`docs/planning/checkov-resolve-plan.md`.

---

## Steps

### 1. Find the latest run

```bash
gh run list --workflow validate-infrastructure.yml --limit 5
```

Pick the most recent completed run on `main` (or the branch in context).
Note its run ID.

### 2. Extract checkov findings

```bash
gh run view <RUN_ID> --log 2>&1 | grep "Security Scan (Checkov)" \
  | grep -E "^Security Scan.*Security Scan.*Z (Check:|PASSED|FAILED|Passed checks|Parsing error|Error parsing)" \
  | sed 's/.*Z //'
```

Collect:

- Summary line: `Passed checks: X, Failed checks: Y, Skipped checks: Z, Parsing errors: N`
- Each `Check: CKV_*` line followed by PASSED or FAILED + resource + file
- Each `Error parsing file` line

### 3. Categorise each finding

For every **FAILED** check, decide:

| Category                         | Criteria                                                     | Action                              |
| -------------------------------- | ------------------------------------------------------------ | ----------------------------------- |
| **Fix**                          | Property missing/wrong in Bicep and easy to add              | Add to Bicep, note file + line      |
| **Skip (by design)**             | Architectural constraint (networking, cost, pipeline access) | Add to `.checkov.yaml` with comment |
| **Parse error (false positive)** | Check on a file that also appears in parsing errors          | Note as false positive, no action   |

### 4. Check `.checkov.yaml` for already-skipped checks

```bash
cat .checkov.yaml
```

Note which checks are already suppressed so the plan doesn't duplicate them.

### 5. Check current Bicep modules for the failing resources

For each FAILED Fix check, read the relevant module file to confirm whether
the property is truly missing or whether it's a checkov parse limitation.

### 6. Write the plan

Write `docs/planning/checkov-resolve-plan.md` with this structure:

```markdown
# Checkov Resolve Plan

Generated: <date>
Run: <GitHub Actions run URL>
Summary: Passed X | Failed Y | Skipped Z | Parse errors N

## To Fix

### CKV*AZURE*<ID> — <check title>

- **Resources affected**: `<resource>` in `infra/modules/<file>.bicep:<lines>`
- **Fix**: <exact Bicep property to add, with example value>

## Already Skipped (`.checkov.yaml`)

| Check         | Reason                              |
| ------------- | ----------------------------------- |
| CKV_AZURE_XXX | <reason from .checkov.yaml comment> |

## Parse Errors (false positives)

Files checkov cannot fully parse — checks on these files are unreliable:

- `infra/modules/<file>.bicep` — <likely cause, e.g. "conditional resource syntax">

## Decided: Skip (not worth fixing)

| Check         | Resource     | Reason          |
| ------------- | ------------ | --------------- |
| CKV_AZURE_XXX | `<resource>` | <justification> |
```

### 7. Report back

Summarise the plan in one paragraph: how many findings, how many fixed vs skipped,
and the path to the written file.
