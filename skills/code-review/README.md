# Code Review

Structured review checklists covering architecture, code structure, concurrency/performance, data structures, security, and integration-code quality/hygiene (URL construction, naming, config vs. secrets).

Cross-cutting — useful regardless of stack. Consider installing globally (`-g`).

## Install

```bash
npx skills add mihailtd/skills/skills/code-review --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/code-review --skill code-review-architecture
```

## Skills (7)

- **[code-review-architecture](code-review-architecture/SKILL.md)** — Guides AI reviewers to evaluate software architecture for functional integrity, structural health, and maintainability.
- **[code-review-code-structure](code-review-code-structure/SKILL.md)** — Guides AI reviewers to evaluate code structure, detect architectural code smells, and recommend cleaner functional-lite designs.
- **[code-review-concurrency-parallelism-performance](code-review-concurrency-parallelism-performance/SKILL.md)** — Guides AI reviewers to evaluate concurrency, parallelism, and performance trade-offs in code.
- **[code-review-data-structures](code-review-data-structures/SKILL.md)** — Guides AI reviewers to evaluate data structure choices for correctness, scalability, performance, and simplicity.
- **[code-review-detect-bad-design](code-review-detect-bad-design/SKILL.md)** — Guides AI reviewers to detect functional design smells and architectural debt in code structure.
- **[code-review-quality-and-hygiene](code-review-quality-and-hygiene/SKILL.md)** — Review URL construction (urllib, not string concatenation), descriptive variable naming, single-BASE_URL-per-integration hygiene, and environment-variable-vs-hardcoded-value decisions. Use when implementing or reviewing API clients, integrations, or agent/automation code that calls HTTP endpoints.
- **[code-review-security](code-review-security/SKILL.md)** — Guides AI reviewers to evaluate code for security posture, threat-resistant design, and secure development lifecycle concerns.
