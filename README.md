<div align="center">

# IA Resources

_A practical collection of reusable skills for AI coding agents._

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-compatible-111827?style=flat-square)](https://agentskills.io)
[![Browse skills](https://img.shields.io/badge/skills.sh-browse-5b5bd6?style=flat-square)](https://skills.sh)

[Get started](#getting-started) • [Browse the catalog](#skill-catalog) • [Repository structure](#repository-structure)

</div>

IA Resources collects focused instructions, references, scripts, and templates that help coding agents handle common engineering tasks consistently. Install the complete catalog or select only the skills that fit your workflow.

## What's included

- Engineering workflows for architecture, domain modeling, TDD, merge conflicts, and pre-commit setup.
- Framework guidance for React, Tailwind CSS, shadcn/ui, Mastra, Vitest, Zod, and Go.
- Agent workflows for QA, peer review, documentation, skill authoring, and implementation handoffs.
- Reusable assets and references that skills load only when a task needs them.
- Starter [`AGENTS.md`](./AGENTS.example.md) and [`CLAUDE.md`](./CLAUDE.example.md) instruction files.

## Getting started

You need [Node.js](https://nodejs.org/) and npm to run the [Skills CLI](https://skills.sh/docs).

List the skills available in this repository:

```bash
npx skills@latest add virb30/ia-resources --list
```

Install skills interactively into the current project:

```bash
npx skills@latest add virb30/ia-resources
```

Install one skill directly:

```bash
npx skills@latest add virb30/ia-resources --skill react
```

Add `--global` to make a skill available across projects:

```bash
npx skills@latest add virb30/ia-resources --skill no-workarounds --global
```

The CLI detects supported agents and lets you choose the installation targets. Once installed, ask your agent to use a skill by name—for example, `Use the tdd skill to implement this change`.

> [!TIP]
> Install only the skills you expect to use. A smaller active set makes skill selection more predictable.

## Skill catalog

### Curated

Reviewed, generally useful skills for regular work.

- [context7](./skills/curated/context7) — Retrieves current, authoritative documentation and code examples for developer technologies.
- [deslop](./skills/curated/deslop) — Removes unnecessary AI-generated patterns and noise from a branch diff.
- [find-skills](./skills/curated/find-skills) — Discovers and installs skills for a requested capability.
- [golang-pro](./skills/curated/golang-pro) — Guides idiomatic Go development, concurrency, microservices, performance, and testing.
- [impl-peer-review](./skills/curated/impl-peer-review) — Runs an independent cross-LLM review of an implemented change.
- [lesson-learned](./skills/curated/lesson-learned) — Extracts reusable engineering lessons from recent Git history.
- [mastra](./skills/curated/mastra) — Provides current guidance for building Mastra agents, tools, workflows, memory, and RAG.
- [no-workarounds](./skills/curated/no-workarounds) — Diagnoses and fixes root causes instead of masking symptoms.
- [qa-execution](./skills/curated/qa-execution) — Runs persona-driven dogfooding sessions through a product's public interfaces.
- [qa-report](./skills/curated/qa-report) — Plans durable QA journeys, scenarios, charters, and bug records.
- [react](./skills/curated/react) — Guides React component architecture, hooks, state, TypeScript, and testing.
- [shadcn](./skills/curated/shadcn) — Applies shadcn/ui and Radix UI composition, styling, and design-token practices.
- [skill-best-practices](./skills/curated/skill-best-practices) — Authors professional skills that follow the agentskills.io specification.
- [spec-peer-review](./skills/curated/spec-peer-review) — Runs an independent cross-LLM review of a specification or design document.
- [tailwindcss](./skills/curated/tailwindcss) — Applies Tailwind CSS v4 conventions for tokens, responsive layouts, and utilities.
- [to-prompt](./skills/curated/to-prompt) — Packages code, issues, and context into an implementation brief for another LLM.
- [vitest](./skills/curated/vitest) — Guides Vitest configuration, tests, mocks, fixtures, filtering, and coverage.
- [writing-agents-md](./skills/curated/writing-agents-md) — Authors concise, high-value `AGENTS.md` and `CLAUDE.md` instruction files.
- [writing-skills](./skills/curated/writing-skills) — Creates, restructures, and debugs agent skills and their metadata.
- [zod](./skills/curated/zod) — Applies Zod validation patterns for safe parsing, schemas, errors, and type inference.

### Evaluating

Skills being assessed or refined before broader use.

- [codebase-design](./skills/evaluating/codebase-design) — Uses deep-module vocabulary to improve interfaces, seams, and testability.
- [create-readme](./skills/evaluating/create-readme) — Creates a concise, comprehensive, and project-specific README.
- [domain-modeling](./skills/evaluating/domain-modeling) — Defines domain terminology, ubiquitous language, and architectural decisions.
- [grill-me](./skills/evaluating/grill-me) — Stress-tests a plan or design through a relentless interview.
- [grill-with-docs](./skills/evaluating/grill-with-docs) — Stress-tests a plan while producing supporting ADRs and glossary entries.
- [grilling](./skills/evaluating/grilling) — Challenges assumptions and tradeoffs in a plan, decision, or idea.
- [improve-codebase-architecture](./skills/evaluating/improve-codebase-architecture) — Finds architecture-deepening opportunities and presents them in a visual report.
- [resolving-merge-conflicts](./skills/evaluating/resolving-merge-conflicts) — Resolves in-progress Git merge and rebase conflicts.
- [setup-pre-commit](./skills/evaluating/setup-pre-commit) — Configures Husky and lint-staged for formatting, type checks, and tests before commits.
- [tdd](./skills/evaluating/tdd) — Implements features and fixes with a red-green-refactor workflow.
- [teach](./skills/evaluating/teach) — Teaches a skill or concept through a structured, workspace-based learning process.
- [to-spec](./skills/evaluating/to-spec) — Turns the current conversation into a specification published to the issue tracker.

### Mine

Custom, memory-enhanced variants maintained in this repository.

- [impl-peer-review-memory](./skills/mine/impl-peer-review-memory) — Reviews implementation diffs and can preserve human corrections as long-term lessons.
- [lesson-memory](./skills/mine/lesson-memory) — Records a supplied mistake, its cause, and the correct approach for future work.

> [!NOTE]
> Skills under `evaluating` and `mine` can evolve more quickly than the curated set. Review their `SKILL.md` before adopting them in shared or automated workflows.

## Repository structure

```text
.
├── skills/
│   ├── curated/       # Reviewed skills
│   ├── evaluating/    # Skills under evaluation
│   └── mine/          # Repository-specific variants
├── skills-lock.json   # Upstream source paths and integrity hashes
├── AGENTS.example.md  # Starter agent instructions
└── CLAUDE.example.md  # Claude Code bridge to AGENTS.md
```

Every skill is rooted at a `SKILL.md` file. Depending on its needs, a skill can also include:

- `references/` for detailed guidance loaded on demand;
- `assets/` for templates and other reusable material;
- `scripts/` for validation or automation;
- `agents/` for agent-specific metadata.

The generated `.agents/` and `.claude/` directories are intentionally ignored. Edit the source under `skills/`, not an installed copy.

## Keeping skills current

Check installed project skills:

```bash
npx skills@latest list
```

Update project-level installations:

```bash
npx skills@latest update --project
```

The committed [`skills-lock.json`](./skills-lock.json) records the upstream repository, source path, and computed hash for imported skills so changes remain auditable.
