# qa-ranger

Give it your project documentation — files, URLs, or a repo. Get back structured test cases ready to submit.

It reads everything you provide, maps the system under test, ranks risk areas by business impact, and writes test cases organized by priority. Works for API projects, UI projects, or both.

---

## Why it works this way

Most test case generators start from a test type checklist. This one starts from domain risk.

- **Domain-risk-first** — reasoning begins with "what does failure cost this business?" not "what test types should I apply?" Priority follows consequence, not convention.
- **Domain Pattern Library** — a versioned knowledge asset that encodes senior QA judgment for seven common domains. The strategist loads it on every run and matches your system against known failure modes, state transitions, and edge cases before writing a single test case.
- **Confidence as the exit criterion** — coverage stops when a senior QA would sign off, not when a test count is hit.

---

## How it works

Three skills run in sequence. Each one produces a file the next one consumes:

```
/qa-ranger [files | URLs | free text | nothing]
    │
    ▼
1. Explorer     → _qa/domain-analysis.md
    │
    ▼
2. Strategist   → _qa/risk-strategy.md
    │
    ▼
3. Writer       → _qa/test-cases.md
                  _qa/coverage-summary.md
```

**1. Explorer** — reads your inputs (files, URLs, free text, or auto-scans the project). Classifies the task as API-only, UI-only, or both. Maps every endpoint, flow, data model, and requirement. Flags ambiguities explicitly.

**2. Strategist** — loads the Domain Pattern Library, matches the system against known domains (billing, auth, inventory, etc.), ranks risk areas by business impact, and selects test techniques. Also mines implied business rules from documentation notes — behaviors not captured in the API spec but inferable from product context, help articles, or business constraints. Each extracted rule must appear as a named test area. Produces a specific brief for the writer.

**3. Writer** — executes the brief. Writes test cases in Gherkin format with YAML metadata (priority, type, technique, automation candidate). Organized by risk priority — P0 first, P3 last. Produces a plain-language coverage summary for stakeholders.

You invoke one command. The three steps run automatically end to end. If no business rule references are found in your inputs, the agent will tell you and suggest re-running with additional context — help articles, product guides, or a task description.

---

## Requirements

[Claude Code](https://claude.ai/code) — Pro, Max, or API access. No other dependencies.

---

## Setup

Clone the repo, then copy the 5 files into a commands directory (`qa-ranger-explorer.md`, `qa-ranger-strategist.md`, `qa-ranger-writer.md`, `qa-ranger-patterns.md`, `qa-ranger.md`):

```bash
git clone https://github.com/aslavchev/qa-ranger.git
```

**This project only:**
```bash
mkdir -p /path/to/your-project/.claude/commands
cp qa-ranger/skills/*.md /path/to/your-project/.claude/commands/
cp qa-ranger/agent/qa-ranger.md /path/to/your-project/.claude/commands/
```

**All your projects:**
```bash
cp qa-ranger/skills/*.md ~/.claude/commands/
cp qa-ranger/agent/qa-ranger.md ~/.claude/commands/
```

The 5 files must all be present — the orchestrator calls the four skills internally. You only ever invoke `/qa-ranger`. The individual skills (`/qa-ranger-explorer`, `/qa-ranger-strategist`, `/qa-ranger-writer`) can also be called directly if you want to re-run a specific stage without restarting the full pipeline.

Add `_qa/` to your `.gitignore` to avoid committing run outputs.

---

## Usage

```
/qa-ranger https://developer.example.com/api/memberships
```

```
/qa-ranger ./docs/api-spec.yaml
Context: [any additional context — task description, requirements, scope notes]
```

```
/qa-ranger
```
Current repo as input — scans README, API specs, route files, and test files automatically.

---

## What you get

Four files in `_qa/`:

| File | What it contains |
|------|-----------------|
| `domain-analysis.md` | System mapped: endpoints, flows, actors, implicit requirements, ambiguities flagged |
| `risk-strategy.md` | Risk areas ranked by business impact, techniques selected with reasoning |
| `test-cases.md` | Test cases in Gherkin format, priority-ordered, grouped by functional area |
| `coverage-summary.md` | Plain-language confidence statement for stakeholders |

This tool produces test planning artifacts. Test execution and automation are separate steps.

---

## Domain Pattern Library

The library is installed as `qa-ranger-patterns.md` alongside the other skills — it is a required file, not baked into the agent. The strategist loads it on every run. Seven domains are included:

- **Billing / Recurring Charges** — charge idempotency, proration edge cases, subscription state transitions, webhook replay
- **Authentication / Authorization** — IDOR, privilege escalation, token lifecycle, session management, JWT edge cases
- **User / Member Management** — soft delete, role assignment, relationship integrity, bulk operations
- **Inventory / Resource Management** — concurrency, race conditions on last unit, reservation expiry, cache staleness
- **Search / Filtering** — pagination correctness, empty results, sort stability, injection via filter parameters
- **File / Media Handling** — MIME spoofing, path traversal, async processing failures, unauthenticated storage access
- **Notifications / Messaging** — duplicate delivery, silent failure, template rendering, rate limiting, dead-letter queues

Each pattern covers: critical failure modes, state transitions, non-obvious edge cases, security surface, performance concerns, and UI surface risks.

New domains can be added using the template at the bottom of `qa-ranger-patterns.md` wherever it is installed. The library grows with each new assignment.

---

## Example

See `examples/api-assignment/` — a gym membership API with billing, auth, and member management domains. Includes all four output files and a coverage gap scenario.

---

## License

MIT — see [LICENSE](LICENSE).
