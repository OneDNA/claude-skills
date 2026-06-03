# claude-skills

Shared Claude Code skills for OneDNA projects. Skills are organized into plugin groups — each group maps to a domain and can be symlinked into `.claude/skills/` in any project repo.

## Structure

```
claude-skills/
  architecture/    # Codebase understanding, diagrams, schema design
  devops/          # CI/CD, security scanning, dependency management
  docs/            # Documentation generation and updates
  frontend/        # UI design, prototyping, design-to-code pipelines
  meta/            # Skill discovery and management
  planning/        # Design reviews, PRDs, decision grilling
  power-bi/        # Power BI semantic models, DAX, reports, deployment
  testing/         # TDD, Playwright, diagnosis, validation
```

## Plugins

### architecture

| Skill | Description |
|---|---|
| `dbml-logical-to-physical` | Convert DBML logical schemas to physical schemas for a target database engine |
| `drawio-diagrams-enhanced` | Create professional draw.io diagrams with PMP/PMBOK templates and shape libraries |
| `improve-codebase-architecture` | Find deepening opportunities informed by domain language and ADRs |
| `learn-codebase` | Structured onboarding into an unfamiliar codebase |
| `zoom-out` | Get a higher-level perspective on how code fits into the bigger picture |

### devops

| Skill | Description |
|---|---|
| `azure-cost-monitor` | Monitor and report on Azure resource costs |
| `checkov-review` | Static security analysis of infrastructure-as-code with Checkov |
| `dependabot-parallel` | Merge multiple Dependabot PRs in parallel batches |
| `dependabot-review` | Review and triage Dependabot pull requests |
| `post-rebase-check` | Integrity check after a rebase to catch silently dropped files |
| `pr-cycle` | Full PR review and iteration loop |

### docs

| Skill | Description |
|---|---|
| `docs-update` | Update project documentation after a PR merges, with nav and ADR suggestions |

### frontend

| Skill | Description |
|---|---|
| `design-to-prod` | Orchestrate Claude Design → Storybook → Production with sign-off gates |
| `frontend-design` | Build UI components, pages, and interfaces following a structured design framework |
| `prototype` | Build throwaway prototypes to validate logic or explore UI variations |

### meta

| Skill | Description |
|---|---|
| `find-skills` | Discover and install agent skills for a given task |

### planning

| Skill | Description |
|---|---|
| `grill-me` | Relentless interview-style review of a plan to resolve every design decision |
| `grill-with-docs` | Stress-test a plan against the existing domain model and update ADRs inline |
| `to-prd` | Turn conversation context into a PRD and publish it to the issue tracker |

### power-bi

| Skill | Description |
|---|---|
| `power-bi-custom-visuals` | Scaffold, build, and package custom `.pbiviz` visuals |
| `power-bi-dax` | Write, execute, and optimize DAX queries and measures |
| `power-bi-deployment` | Export/import TMDL/TMSL, manage model lifecycle and version control |
| `power-bi-diagnostics` | Troubleshoot performance, trace queries, verify pbi-cli environment |
| `power-bi-docs` | Auto-document semantic models with data dictionaries and catalogs |
| `power-bi-filters` | Configure page and visual filters (TopN, date, categorical) |
| `power-bi-modeling` | Create tables, columns, measures, relationships, and hierarchies |
| `power-bi-pages` | Manage report pages, bookmarks, visibility, and drillthrough |
| `power-bi-partitions` | Manage partitions, M expressions, and data sources |
| `power-bi-report` | Scaffold, validate, and preview PBIR reports |
| `power-bi-security` | Configure RLS roles, object-level security, and perspectives |
| `power-bi-themes` | Apply themes, conditional formatting, and styling |
| `power-bi-visuals` | Add, bind, update, and bulk-manage report visuals |

### testing

| Skill | Description |
|---|---|
| `diagnose` | Disciplined diagnosis loop for hard bugs and performance regressions |
| `playwright-cli` | Automate browser interactions and work with Playwright tests |
| `playwright-local-validation` | Run and validate Playwright tests in a local environment |
| `tdd` | Test-driven development with red-green-refactor loop |
| `verify-before-done` | End-to-end validation before claiming a task complete |

## Usage

Symlink a plugin group (or individual skill) into your project:

```bash
# Entire plugin group
ln -s /path/to/claude-skills/testing .claude/skills/testing

# Individual skill
ln -s /path/to/claude-skills/architecture/zoom-out .claude/skills/zoom-out
```

Or add this repo as a git submodule:

```bash
git submodule add https://github.com/OneDNA/claude-skills.git claude-skills
git submodule update --init --recursive
```
