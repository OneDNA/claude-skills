---
name: docs-update
description: >
  Update project documentation after a PR merges. Always starts with the
  planning section, then maps changed files to affected docs, edits all
  sections in parallel where possible, validates every page with an
  independent reviewer agent that checks correctness and tests code blocks,
  updates mkdocs.yml nav if new pages were added, and optionally suggests
  new ADRs or sections. Opens a docs/<description> branch + PR.
tools: Bash, Read, Edit, Write, Glob, Grep, Agent
---

# Docs Update

Run this skill after every PR merges, as the last step before recording the
outcome. Follow every phase in order. Be explicit about what you are adding
AND what you are removing.

---

## Phase 0 — Always update the planning section first

Before touching any architecture or CI/CD docs, update all three planning
files to reflect the merged work.

Read all three files in parallel, then:

1. **`wip.md`** — add every merged PR to the "Recently Completed" table.
   Format: `| <short task description> | #<PR> | <YYYY-MM-DD> |`
   Keep rows sorted newest-first. Trim rows older than ~6 months to the
   archived section (or delete if already in roadmap Completed).

2. **`roadmap.md`** — move completed items from Backlog/To Do columns into
   the Done column of the Kanban, and add a matching bullet to the
   `## Completed` section at the bottom.

3. **`implementation-history.md`** — add a new dated section at the top for
   this session's work. Format:

   ```markdown
   ## YYYY-MM-DD — <short session title>

   **PRs:** #N, #M
   **Branch:** <branch name>

   ### <Feature/Fix name> (PR #N)

   <2–4 sentence description of what changed and why. Include key files changed,
   architectural decisions made, and any non-obvious implementation details.>
   ```

   Group PRs by logical theme if multiple landed in the same session. Each entry
   should be detailed enough to reconstruct what happened without reading the diff.
   Keep the most recent section at the top (newest-first).

State: "Planning updated: added `<N>` rows to wip.md, moved `<M>` items in roadmap.md, added `<K>` sections to implementation-history.md."

---

## Phase 1 — Map changed files to affected docs

List files changed in the merged PR:

```bash
git diff --name-only <base-sha> <head-sha>
# or for the most recent merge:
git diff --name-only HEAD~1 HEAD
```

Use this mapping to identify which docs files need updating:

| Changed file pattern                         | Affected docs                                                                        |
| -------------------------------------------- | ------------------------------------------------------------------------------------ |
| `client/**`, `vite.config.ts`, `nginx*.conf` | `docs/architecture/frontend.md`                                                      |
| `api/**`, `requirements.txt`                 | `docs/architecture/backend.md`, `docs/api/endpoints.md`                              |
| `infra/**`                                   | `docs/infrastructure/bicep-structure.md`, `docs/infrastructure/resource-overview.md` |
| `.github/workflows/deploy-*.yml`             | `docs/ci-cd/pipeline-overview.md`                                                    |
| `.github/workflows/validate-*.yml`           | `docs/ci-cd/pipeline-overview.md`                                                    |
| `docs/**`, `mkdocs.yml`                      | Skip — docs changes are self-describing                                              |
| `.claude/**`                                 | Skip — internal tooling, not user-facing docs                                        |

If no files in the table are affected beyond planning: state "No further docs
update needed." and skip to Phase 4.

State: "Affected docs: <list of files>"

---

## Phase 2 — Read all affected docs files

Read every affected docs file in full **in parallel** before editing anything.

For each file, identify:

1. **Content to update** — sections that describe behaviour changed by the PR
2. **Content to remove** — sections that are now incorrect or obsolete (be explicit; list line ranges)
3. **Content to add** — new sections or notes the PR introduces

State for each file:

- Update: `<what>`
- Remove: `<what>` (line ~N–M)
- Add: `<what>`

---

## Phase 3 — Edit docs files in parallel

Where files are independent of each other, launch edits **in parallel** using
the Agent tool (subagent_type: general-purpose) — one agent per file. Each
agent receives the full file content, the list of changes from Phase 2, and
the editing rules below.

For files that share content (e.g. cross-references), edit sequentially to
avoid contradictions.

### Adding content

- Add new sections directly below the most closely related existing section.
- Match the heading level of adjacent sections.
- Keep prose concise — prefer tables and code blocks over paragraphs.

### Updating content

- Edit in place. Do not move content to a new location unless necessary.
- Update code examples to reflect the current API, parameters, and URLs.

### Markdown tables

- Do **not** re-align table column padding after edits. Write cell content
  without trailing spaces (compact style). MkDocs and GitHub both render
  markdown tables correctly regardless of column alignment.
- Ignore linter warnings about table pipe alignment (MD060).

### Removing content

- **Explicitly delete** any line or section that is now incorrect or obsolete.
  Do not leave it commented-out or marked as deprecated.
- After removing, check for orphaned references (links, table rows, list items)
  that pointed to the removed content. Remove those too.
- State what you removed and why: "Removed `<section>` — superseded by `<new behaviour>`."

### Frontmatter changelog (required on every modified file)

Every docs file that is modified must have a YAML frontmatter block at the very
top. If the file has no frontmatter yet, add one. If it already has one, append
a new entry to the `changes` list.

```yaml
---
last_updated: YYYY-MM-DD
changes:
  - date: YYYY-MM-DD
    prs: [<number>, ...]
    summary: "<one sentence: what was added or changed>"
    removed: "<one sentence: what was explicitly deleted, or omit if nothing removed>"
---
```

Rules:

- `last_updated` is always today's date (the date you run this skill).
- `prs` is the list of PR numbers that caused this docs change.
- `summary` covers additions and corrections.
- `removed` is required whenever you delete content from the file. If nothing
  was removed, omit the `removed` key entirely.
- Prepend the frontmatter block before the existing first line (`# Title`).
  Do not alter the heading itself.

After each file edit, state: "Edited `<file>`: added `<X>`, removed `<Y>`."

---

## Phase 4 — Validate each edited page with a reviewer agent

After all edits are complete, launch **one reviewer agent per modified file**
in parallel (subagent_type: general-purpose). Each agent must:

### 4a — Paragraph-level correctness check

Read the edited file and the source code it documents. For every prose
paragraph, verify:

- All claims match the current source code / config / infra files
- No stale version numbers, endpoint paths, env var names, or resource names
- No references to removed features or old behaviour
- Cross-links to other docs pages resolve to existing files

Report any issue as: `[STALE] <file>:<line> — <what is wrong> (should be: <correct value>)`

If no issues: `[OK] <file> — all paragraphs correct`

### 4b — Code block testing

For every fenced code block in the file:

1. Classify the block:
   - `bash` / `shell` — runnable
   - `yaml` / `bicep` / `typescript` / `python` — syntax-checkable
   - `dockerfile` — checkable
   - `text` / `output` / display-only — skip

2. For **runnable** `bash` blocks:
   - Run the command in a sandboxed shell (`bash -c "<command>"` or equivalent)
   - Skip commands that require Azure credentials, live deployments, or external
     network calls (mark as `[SKIPPED — requires live env]`)
   - Skip destructive commands (`az group delete`, `docker rm -f`, `git reset --hard`)
   - Report: `[PASS]`, `[FAIL: <error>]`, or `[SKIPPED — <reason>]`

3. For **YAML** blocks: run `python -c "import yaml; yaml.safe_load(open('/dev/stdin'))"` on the block
4. For **TypeScript** snippets: run `npx tsc --noEmit --strict` on a temp file if the project tsconfig is reachable
5. For **Python** snippets: run `python -m py_compile` on a temp file

Report every tested block with its result. Aggregate at the end:

```
Code block test summary for <file>:
  PASS:    <N>
  FAIL:    <N>  ← list each failure
  SKIPPED: <N>
```

### 4c — Fix failures

If the reviewer finds `[STALE]` issues or `[FAIL]` code blocks, fix them
**in place** before proceeding. Re-run the affected block after fixing to
confirm it passes.

After all reviewers complete, state:
"Validation complete: `<N>` files reviewed, `<M>` issues fixed, `<K>` blocks
tested (`<P>` pass / `<F>` fail / `<S>` skipped)."

---

## Phase 5 — Update navigation

Navigation is defined in **`docs/.nav.yml`** (managed by the `awesome-nav`
MkDocs plugin — not in `mkdocs.yml` itself). Read `docs/.nav.yml` and compare
its `nav:` tree against the actual files in `docs/`.

**awesome-nav syntax** (`docs/.nav.yml`):

```yaml
nav:
  - Home: index.md
  - Section Title:
      - path/to/page.md # title inferred from H1
      - Page Title: path/to/page.md # explicit title
      - Subsection:
          - sub/page.md
```

Update `docs/.nav.yml` if any of the following are true:

- A new `.md` file was created in Phase 3 or approved in Phase 6 and is not
  yet listed
- A file was renamed or moved and the old path is still referenced
- A file was deleted and its entry was not removed

**Rules:**

- Place new entries under the most logically appropriate section — match the
  directory structure (`docs/architecture/` → under Architecture, etc.)
- Preserve the existing order within each section; append new entries at the
  end of the relevant section unless a different position is clearly more logical
- Do **not** reorder or restructure the entire nav — only touch the entries
  that need changing
- Also remove entries for files that no longer exist on disk
- If no nav changes are needed: state "Nav up to date — no changes needed."

After editing `docs/.nav.yml`, verify locally if the docs container is running:

```bash
curl -s -o /dev/null -w "Docs: HTTP %{http_code}\n" http://localhost:8080/
docker-compose logs docs --tail=10
```

A non-500 response + no new `WARNING` lines confirms MkDocs accepted the nav.

State: "Nav updated: added `<entries>` / removed `<entries>`." or "Nav up to date."

---

## Phase 6 — Suggest new ADRs or sections (optional)

Review the changes introduced by the PR and ask: **did this PR introduce a
decision or pattern that is not yet captured anywhere in the docs?**

Only raise a suggestion if one of these is true:

- A significant architectural decision was made that has long-term consequences
  (e.g. choosing a new auth mechanism, switching a data store, adopting a new
  runtime pattern) — suggest a new ADR in `docs/architecture/decisions/`
- A new cross-cutting concern emerged (e.g. a new environment variable
  convention, a new deployment pattern, a new testing strategy) that would
  benefit from its own section rather than being buried in an existing page
- An existing section has grown large enough that it should be split into a
  dedicated page

**Do not suggest** ADRs or sections for:

- Routine feature additions (new endpoint, new component, new config value)
- Bug fixes
- Changes already well-documented by the edits in Phase 3

If nothing qualifies: state "No new ADRs or sections needed." and move on.

If something qualifies, output a suggestion block — do **not** create the
file unless the user confirms:

```
## Suggested ADR: <short title>
File: docs/architecture/decisions/<NNN>-<slug>.md
Reason: <one sentence why this decision warrants an ADR>
Key points to capture:
- <bullet>
- <bullet>

## Suggested new section: <title>
Location: <file> → after section "<heading>"
Reason: <one sentence>
```

Wait for user approval before writing any new files.

---

## Phase 7 — Commit docs changes

### If run during the PR cycle (on a feature branch, before merge)

Commit the docs directly to the current branch — they ship with the code PR.
Include any ADR or new section files if the user approved them in Phase 5:

```bash
git add docs/
git commit -m "$(cat <<'EOF'
docs(<scope>): update docs for <feature summary>

- <bullet: what was added>
- <bullet: what was removed>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
git push
```

State: "Docs committed to current branch."

### If run after a PR merges (from main)

Create a separate docs branch and PR:

```bash
git checkout main && git pull origin main
git checkout -b docs/<short-description>

git add docs/
git commit -m "$(cat <<'EOF'
docs(<scope>): update docs after <PR title>

- <bullet: what was added>
- <bullet: what was removed>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"

git push -u origin docs/<short-description>

gh pr create \
  --title "docs(<scope>): update docs after <PR title>" \
  --body "$(cat <<'EOF'
## Summary

Updates docs to reflect changes merged in <PR numbers>.

### Added
- <bullet>

### Removed
- <bullet> — reason: <why it was removed>

### Validation
- <N> pages reviewed by independent agent
- <P> code blocks passed / <F> failed / <S> skipped

### Frontmatter
Each modified file has an updated `changes` entry in its YAML frontmatter.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

State: "Docs PR opened: <URL>"

---

## Execution rules

- **Planning section is always updated first**, regardless of what changed.
- **During PR cycle** (feature branch, before merge): commit docs to the same branch — they go in the same PR.
- **After PR merges** (running from main): docs always get their own `docs/` branch and PR.
- Never mark a doc section as "deprecated" — either update it or delete it.
- If a doc file has conflicting information in two places, fix both. Leave no contradictions.
- If you are unsure whether content is still accurate, check the source code before deciding whether to keep, update, or remove it.
- Parallelize Phase 3 edits and Phase 4 validations using the Agent tool wherever files are independent.
- `planning/` files are updated in Phase 0, not via the file-mapping table in Phase 1. Phase 0 covers **all three**: `wip.md`, `roadmap.md`, and `implementation-history.md`. Never skip implementation-history.md.
- `.claude/` files are internal tooling — never included in user-facing docs updates.
