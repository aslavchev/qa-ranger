# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`qa-ranger` is a Claude Code agent that converts unstructured QA assignment inputs (API docs, URLs, repos, business context) into structured, submission-ready test case markdown files. It produces test planning artifacts — test cases and coverage analysis. Test execution and automation are out of scope and handled as separate steps. It is itself a Claude Code agent definition — not a traditional code project. There are no build, lint, or test commands.

## Architecture: 4-Phase Skill Pipeline

The agent orchestrates three skills in sequence, with an orchestrating agent on top:

```
agent/qa-ranger.md          ← orchestrator (Phase 4) — invocation entry point
  │
  ├─► skills/qa-ranger-explorer.md     (Phase 1) — reads inputs, classifies task type
  │         └─► produces: _qa/domain-analysis.md
  │
  ├─► skills/qa-ranger-strategist.md   (Phase 2) — reasons about coverage
  │         ├─► consumes: _qa/domain-analysis.md
  │         ├─► loads:    skills/qa-ranger-patterns.md  (domain knowledge library)
  │         └─► produces: _qa/risk-strategy.md
  │
  └─► skills/qa-ranger-writer.md       (Phase 3) — writes test cases
            ├─► consumes: _qa/risk-strategy.md
            └─► produces: _qa/test-cases.md, _qa/coverage-summary.md
```

`_qa/` is the default output directory (gitignored per run).

## Key Design Principles

**Explorer** — auto-detects input mode (files, URLs, both, or scan current project). Always accepts free text alongside other inputs. Flags ambiguities explicitly rather than silently skipping them. Classifies task as API-only, UI-only, or Both — this drives the entire downstream strategy.

**Strategist** — domain-risk-first reasoning (not ISTQB-first). Uses the Domain Pattern Library (`skills/qa-ranger-patterns.md`) as its primary knowledge source. Maps coverage to Agile Testing Quadrants (Crispin/Gregory): Q1+Q4 dominate for API-only, Q2+Q3 for UI-only, all quadrants for Both. Must explicitly state domain match confidence and flag unknown domains.

**Domain Pattern Library** (`skills/qa-ranger-patterns.md`) — a versioned, expandable knowledge asset separate from the skill prompt. Each pattern covers: critical failure modes, state transitions, non-obvious edge cases, security surface, performance concerns, UI surface risks, and quadrant weight. The `UI surface` field contains domain-specific failure modes that only manifest at the interface layer; the strategist applies it automatically when task type is UI-only or Both. Initial domains: billing/recurring charges, auth/authorization, inventory/resource management, user/member management, search/filtering, file/media handling, notifications/messaging. Grows with each new assignment.

**Writer** — produces test cases in hybrid Gherkin/BDD format with YAML front-matter per case (id, title, type, level, priority P0–P3, technique, automation_candidate). Organized by risk priority (P0 first), grouped by functional area — never by HTTP method or UI screen. Coverage summary is written in plain language readable by non-QA stakeholders.

## Output Format

Test case structure:
```yaml
---
id: TC-{module}-{number}
title: "Descriptive title"
type: functional|security|performance|usability|compatibility
level: integration|system|acceptance
priority: P0|P1|P2|P3
technique: boundary|equivalence|state-transition|exploratory
automation_candidate: yes|no|maybe
---
Given / When / Then scenario
```

Priority scale: P0 = system-breaking/blocking, P1 = core functionality, P2 = important but not blocking, P3 = edge cases.

## Skill Dependency Order

Each skill depends on the previous one's output. This is the data flow, not a build to-do list — all skills are already implemented:

1. `skills/qa-ranger-explorer.md` — no dependencies, entry point
2. `skills/qa-ranger-strategist.md` + `skills/qa-ranger-patterns.md` — consumes explorer output
3. `skills/qa-ranger-writer.md` — consumes strategist output
4. `agent/qa-ranger.md` — orchestrates all three in sequence

## Setup (for new users)

Skills and the agent are Claude Code custom commands. Copy the 5 files to a commands directory to register them — no restart needed.

**One project only** — copy into the project you want to test:
```bash
mkdir -p /path/to/your-project/.claude/commands
cp skills/*.md /path/to/your-project/.claude/commands/
cp agent/qa-ranger.md /path/to/your-project/.claude/commands/
```

**All projects** — install once, available everywhere:
```bash
cp skills/*.md ~/.claude/commands/
cp agent/qa-ranger.md ~/.claude/commands/
```

Add `_qa/` to `.gitignore` in the target project to avoid committing run outputs:
```bash
echo "_qa/" >> .gitignore
```

## Current State

All phases complete. The full pipeline is implemented and ready to use:

- `skills/qa-ranger-explorer.md` — Phase 1
- `skills/qa-ranger-strategist.md` — Phase 2
- `skills/qa-ranger-patterns.md` — Domain Pattern Library v1.1 (7 domains, API + UI surface coverage)
- `skills/qa-ranger-writer.md` — Phase 3
- `agent/qa-ranger.md` — Phase 4 orchestrator
- `examples/api-assignment/` — complete worked example (5 pipeline artifacts)
- `README.md` — project documentation
