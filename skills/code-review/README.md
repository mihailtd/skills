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

## Skills (14)

- **[code-review-architecture](code-review-architecture/SKILL.md)** — Guides AI reviewers to evaluate software architecture for functional integrity, structural health, and maintainability.
- **[code-review-code-structure](code-review-code-structure/SKILL.md)** — Guides AI reviewers to evaluate code structure, detect architectural code smells, and recommend cleaner functional-lite designs — including flagging commented-out code, likely-dead code, duplicated defensive validation, and settled feature flags for deletion.
- **[code-review-concurrency-parallelism-performance](code-review-concurrency-parallelism-performance/SKILL.md)** — Guides AI reviewers to evaluate concurrency, parallelism, and performance trade-offs in code.
- **[code-review-data-structures](code-review-data-structures/SKILL.md)** — Guides AI reviewers to evaluate data structure choices for correctness, scalability, performance, and simplicity.
- **[code-review-detect-bad-design](code-review-detect-bad-design/SKILL.md)** — Guides AI reviewers to detect functional design smells and architectural debt in code structure.
- **[code-review-quality-and-hygiene](code-review-quality-and-hygiene/SKILL.md)** — Review URL construction (urllib, not string concatenation), descriptive variable naming, single-BASE_URL-per-integration hygiene, and environment-variable-vs-hardcoded-value decisions. Use when implementing or reviewing API clients, integrations, or agent/automation code that calls HTTP endpoints.
- **[code-review-naming-fundamentals](code-review-naming-fundamentals/SKILL.md)** — Evaluating identifier names using cognitive-science research: why names matter disproportionately, how formatting supports short-term memory while word choice activates long-term memory, and why naming should be judged at review time, not while coding.
- **[code-review-naming-word-choice](code-review-naming-word-choice/SKILL.md)** — Applying research findings to specific naming choices: full words beat abbreviations/single letters, most single letters carry no shared meaning, camelCase edges out snake_case on accuracy, and poor naming correlates with higher bug density.
- **[code-review-naming-consistency](code-review-naming-consistency/SKILL.md)** — Enforcing naming consistency via "name molds" (the structural pattern a name follows) and Feitelson's three-step model (select concepts, choose words with a project lexicon, construct the name using a consistent mold) — for teams where the same concept gets named a different way every time.
- **[code-review-cognitive-load-smells](code-review-cognitive-load-smells/SKILL.md)** — Flagging Fowler-catalog code smells (long parameter lists, primitive obsession, data clumps, shotgun surgery, and more) by explaining the specific cognitive-load mechanism each one triggers — working-memory overload, failed chunking, or mischunking — plus the Paas Scale as a quick self-assessment technique.
- **[code-review-linguistic-antipatterns](code-review-linguistic-antipatterns/SKILL.md)** — Detecting code where a name's implied behavior contradicts what the code actually does (methods that do more/less/the opposite of what they say, identifiers that claim to hold more/less/the opposite of what they contain), using Arnaoudova's six-category framework.
- **[code-review-clean-architecture-boundaries](code-review-clean-architecture-boundaries/SKILL.md)** — Catching Clean Architecture Dependency Rule violations and layer-structure drift: an entity with an infrastructure-typed field, an inward layer importing outward, a controller/presenter doing more than translation, a rogue top-level folder, a "screaming" structure that reveals its framework instead of its business purpose.
- **[code-review-clean-architecture-functional-style](code-review-clean-architecture-functional-style/SKILL.md)** — Catching OOP creep in Clean Architecture code specifically: an ABC where a `Callable` type belongs, a use-case/controller/service class with one method, a mutating entity method instead of a pure transition function, `Mock(spec=SomeABC)` where a stand-in function belongs.
- **[code-review-security](code-review-security/SKILL.md)** — Guides AI reviewers to evaluate code for security posture, threat-resistant design, and secure development lifecycle concerns.
