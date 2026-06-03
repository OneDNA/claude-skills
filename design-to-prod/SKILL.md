---
name: design-to-prod
description: Orchestrates the Claude Design → Storybook → Production pipeline with user sign-off gates at each phase. Accepts an optional Claude Design URL. At any phase the user can say "back to design" to restart from Phase 1.
user-invocable: true
---

# Design → Storybook → Production Pipeline

Run this pipeline when given a Claude Design URL or a free-form component request. Each phase ends with a sign-off gate — do not proceed until the user approves.

---

## Input

The skill receives one of:

- A Claude Design URL: `https://api.anthropic.com/v1/design/h/<id>`
- A plain-text description of what to build (skip to Phase 2)

---

## Phase 1 — Design Review (optional, runs when a URL is provided)

**Goal**: understand exactly what needs to be built before touching any code.

### Steps

<!-- markdownlint-disable MD029 -->

1. **Fetch the design bundle**

```bash
curl -s "https://api.anthropic.com/v1/design/h/<id>" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  --output /tmp/design.tar.gz
mkdir -p /tmp/design-bundle && tar -xzf /tmp/design.tar.gz -C /tmp/design-bundle
ls /tmp/design-bundle
```

2. **Read the README** — always the first file in the bundle; it names every deliverable.

3. **Read all `chat*.md` transcripts** — the design decisions are in there.

4. **Read any HTML/CSS preview files** — these show final visual intent.

5. **Produce an implementation plan** as a numbered list:
   - Component name (file path)
   - What changes (new file / modify existing)
   - Visual/interaction spec in ≤ 3 bullet points
   - Any new Tailwind classes, tokens, or CSS keyframes needed
   - Storybook stories needed (story names + play function outline)
   - Unit tests needed

<!-- markdownlint-enable MD029 -->

### Sign-off gate

Output the plan, then stop and write:

> **Phase 1 complete.** Does this plan look right, or should I adjust anything before moving to Storybook?

Wait for the user to respond with one of:

- **Approval** ("looks good", "yes", "proceed") → continue to Phase 2
- **Adjustment request** → revise the plan and re-present it; repeat until approved
- **"back to design"** → re-read the bundle, adjust interpretation, re-present plan

---

## Phase 2 — Storybook Implementation

**Goal**: validate the design in isolation before touching production component code.

### Rules

- Write or update stories in `client/src/stories/`
- Every new component gets a story file; existing components get new stories added
- Each story must have a `play` function that asserts the key visible state (text, role, class, src)
- Use `FeatureFlagsContext.Provider` when the component reads feature flags
- Use `ThemeProvider` (wrapped in `withTheme("light"|"dark")` decorator) when the component calls `useTheme`
- Use `Router` from `wouter` when the component contains `Link`
- Story names follow the pattern: `Default`, `DarkMode`, `<VariantName>`, `<FeatureName>Dark`

### Docs block requirement

Every story file **must** include a `docs` block at two levels:

**Component level** (in `meta.parameters`):

```ts
parameters: {
  docs: {
    description: {
      component: "One paragraph: what the component is, its key variants, and any motion/interaction behaviour worth calling out.",
    },
  },
},
```

**Story level** (on each named export that isn't trivially obvious from its name):

```ts
export const SomethingSpecific: Story = {
  parameters: {
    docs: {
      description: {
        story:
          "One sentence: what makes this variant different and what to look for.",
      },
    },
  },
  // ...
};
```

The `Default` story may omit a story-level description if the component description already covers it. All other stories — especially feature-flag variants, dark mode, and animated states — must have one.

### Steps

1. Write/update story files — one file per component, docs blocks included
2. Rebuild Storybook: `docker-compose restart storybook` (volume-mounted, no rebuild needed)
3. Wait ~20s then confirm Storybook is serving: `curl -s -o /dev/null -w "%{http_code}" http://localhost:6006/`
4. List every story URL for the user to review

### Sign-off gate

Output the story URLs, then stop and write:

> **Phase 2 complete.** Stories are live at the URLs above. Approve to move to production code, or say "back to design" to revisit the plan.

Wait for the user to respond:

- **Approval** → continue to Phase 3
- **Story revision request** → update stories, restart storybook, re-present URLs
- **"back to design"** → return to Phase 1

---

## Phase 3 — Production Implementation

**Goal**: implement the approved design in production component code with full test coverage.

### Rules

- One commit per logical unit (component, test file, CSS block) — never batch unrelated changes
- Check branch with `git branch --show-current` before every commit
- Run `npm run typecheck` (or `npx tsc --noEmit`) in `client/` before committing
- Run the full test suite `npx vitest run` before committing
- After production code changes: rebuild frontend image and restart container
  - `docker-compose build --no-cache frontend && docker-compose up -d frontend`
- Do not change any story file in this phase unless a story import breaks due to a renamed export

### Steps per component

1. Implement the component in `client/src/components/`
2. Update any backend schema / feature flag if the component depends on a new flag
3. Write/update unit tests in `client/src/__tests__/`
4. Run `npx vitest run` — all tests must pass
5. Commit with a descriptive message: `feat(<scope>): <what>`
6. Rebuild and restart frontend container
7. Verify at `http://localhost/`

### Backend handoff check

Before writing Playwright tests, assess whether the feature requires backend work that is not yet done:

- New API endpoint or route
- New database model or migration
- New feature flag that must be seeded in Azure App Configuration
- Auth or permission changes

If **any** of the above apply and the backend work has not been completed in this session, **stop** and produce a handoff document at `docs/planning/handoff-<feature>.md`:

```markdown
# Backend handoff — <feature>

## What the frontend expects

- Endpoint / flag / model needed (exact path, method, schema)

## What still needs to happen

- [ ] numbered backend tasks

## Acceptance criteria for Playwright

- Conditions that must be true before E2E tests can be written

## Branch / context

- Current branch: <branch>
- Relevant frontend files: <list>
```

Then stop and write:

> **Backend work required before E2E tests.** Handoff doc written to `docs/planning/handoff-<feature>.md`. Complete the backend tasks listed there, then re-invoke `/design-to-prod` to continue from the Playwright step.

If no backend work is outstanding, continue directly to Playwright tests.

### Playwright E2E tests

Write or update tests in `e2e/` covering the golden path and the most important edge case for each new user-facing behaviour:

- Use `context.route()` to mock API responses for error/loading states
- Use `page.waitForSelector` or `expect(locator).toBeVisible()` — never fixed `waitForTimeout`
- Auth: follow the existing `global-setup.ts` pattern (`X-ZUMO-AUTH` header) — do not add new auth logic
- Test file naming: `e2e/<feature>.spec.ts`
- Run locally: `npx playwright test e2e/<feature>.spec.ts --headed` (confirm command works before committing)
- Commit test file separately from component code: `test(<scope>): add E2E for <feature>`

### Sign-off gate

After all components and tests are done:

> **Phase 3 complete.** Changes are live at <http://localhost/>. Summary of commits:
> (list each commit hash + one-line message)
>
> Approve to close out, or say "back to design" to revisit the plan.

---

## Escape Hatch — "back to design"

If the user says "back to design" at any point, return to Phase 1 Step 5 and re-present a revised implementation plan based on the feedback received. Do not discard Storybook or production code already written — update it incrementally.

---

## Checklist (internal — verify before each sign-off gate)

- [ ] All TypeScript errors resolved (`npx tsc --noEmit` clean)
- [ ] All vitest tests passing
- [ ] No emoji in any file
- [ ] No direct commits to `main`
- [ ] Feature flag changes touch: frontend context type + defaultFlags + backend schema + router GET+PATCH + `_FALLBACK_ENV_VARS`
- [ ] Storybook stories include dark mode variant if component has dark styles
- [ ] Play functions assert something visual (role, text, class, or src) — not just `toBeTruthy()`
- [ ] Every story file has a component-level `docs.description.component` string
- [ ] Every non-obvious story export has a story-level `docs.description.story` string
- [ ] Backend handoff check done before writing Playwright tests
- [ ] Playwright tests committed separately from component code; command verified locally before commit
