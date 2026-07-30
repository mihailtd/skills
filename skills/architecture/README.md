# Architecture

Enterprise/solution architecture practices: diagramming techniques (domain storytelling, roadmaps, stakeholder maps, portfolio diagrams), risk management, and resilient system design.

Cross-cutting — not tied to a specific codebase or stack. Most teams install this globally (`-g`) rather than per-project.

## Install

```bash
npx skills add mihailtd/skills/skills/architecture --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/architecture --skill architecture-building-with-failure-in-mind
```

## Skills (23)

- **[architecture-building-with-failure-in-mind](architecture-building-with-failure-in-mind/SKILL.md)** — Guides the design of systems that handle dependency failures through graceful degradation, graceful backoff, and failing early.
- **[architecture-diagrams-application-information-concept-diagram](architecture-diagrams-application-information-concept-diagram/SKILL.md)** — Guide for creating an Application/Information Concept Diagram or matrix that cross-maps application components to the information concepts they expose and highlights security and data usage risks.
- **[architecture-diagrams-application-portfolio-diagram](architecture-diagrams-application-portfolio-diagram/SKILL.md)** — Guide for creating an Application Portfolio Diagram that maps the enterprise application landscape, supports portfolio management, and highlights lifecycle, redundancy, hosting location, and criticality.
- **[architecture-diagrams-business-process-map](architecture-diagrams-business-process-map/SKILL.md)** — Guide for creating a Business Process Map / Business Process Diagram that links business processes to their functions, owners, and architectural impact.
- **[architecture-diagrams-business-roles-map](architecture-diagrams-business-roles-map/SKILL.md)** — Guide for creating a Business Roles Map that captures governance, accountabilities, business actors, and role responsibilities in an enterprise architecture context.
- **[architecture-diagrams-domain-modelling](architecture-diagrams-domain-modelling/SKILL.md)** — Guides the creation of domain story diagrams using a pictographic language that captures actors, work objects, activities, and collaboration clearly without branching logic.
- **[architecture-diagrams-goal-initiative-diagram](architecture-diagrams-goal-initiative-diagram/SKILL.md)** — Guide for creating a Goal/Initiative Diagram that connects strategic goals to concrete work packages and initiatives for execution planning.
- **[architecture-diagrams-goal-objective-diagram](architecture-diagrams-goal-objective-diagram/SKILL.md)** — Guide for creating a Goal/Objective Diagram or matrix that maps strategic goals to measurable objectives and clarifies how strategy is translated into execution.
- **[architecture-diagrams-information-concept-business-process-diagram](architecture-diagrams-information-concept-business-process-diagram/SKILL.md)** — Guide for creating an Information Concept/Business Process Diagram or matrix that cross-maps information concepts to business processes and highlights where information circulates across the organization.
- **[architecture-diagrams-information-map](architecture-diagrams-information-map/SKILL.md)** — Guide for creating an Information Map that models key business information concepts, their definitions, categories, and relationships for enterprise architecture.
- **[architecture-diagrams-organization-map](architecture-diagrams-organization-map/SKILL.md)** — Guide for creating an Organization Map diagram that visualizes internal business units, external partners, and the working relationships between them. Use this skill when you need an enterprise ecosystem view instead of a simple hierarchical org chart.
- **[architecture-diagrams-risk-matrix](architecture-diagrams-risk-matrix/SKILL.md)** — Guides the construction of a living risk matrix spreadsheet for visualizing and tracking architecture risks by likelihood, severity, mitigation, and monitoring status.
- **[architecture-diagrams-roadmap](architecture-diagrams-roadmap/SKILL.md)** — Guide for creating an Architecture Roadmap that plots work packages over time, connects them to goals and objectives, and surfaces execution timing, ownership, cost, and progress.
- **[architecture-diagrams-spider-charts](architecture-diagrams-spider-charts/SKILL.md)** — Guide for creating spider charts (progress charts) that monitor architecture implementation progress across focus areas or deliverables.
- **[architecture-diagrams-stakeholder-map](architecture-diagrams-stakeholder-map/SKILL.md)** — Guide for creating a Stakeholder Map that captures stakeholders, their classifications, concerns, and the architecture deliverables they need.
- **[architecture-diagrams-technology-application-function-map](architecture-diagrams-technology-application-function-map/SKILL.md)** — Guide for creating a Technology/Application Function Map that aligns underlying technology services with the application functions they support.
- **[architecture-diagrams-technology-portfolio-diagram](architecture-diagrams-technology-portfolio-diagram/SKILL.md)** — Guide for creating a Technology Portfolio Diagram that inventories technology components, hardware, infrastructure software, and their hosting and vendor details for architecture lifecycle management.
- **[architecture-diagrams-work-package-portfolio-map](architecture-diagrams-work-package-portfolio-map/SKILL.md)** — Guide for creating a Work Package Portfolio Map that links initiatives and work packages to strategic goals and objectives for execution planning.
- **[architecture-domain-storytelling](architecture-domain-storytelling/SKILL.md)** — Guides collaborative domain storytelling as an architectural practice for capturing business scenarios, domain concepts, and shared understanding across teams.
- **[architecture-functional-domain-driven-implementations](architecture-functional-domain-driven-implementations/SKILL.md)** — Guides the implementation of domain models in a functional style, mapping domain storytelling work objects to types and activities to functions for safer state transitions.
- **[architecture-reduce-risk](architecture-reduce-risk/SKILL.md)** — Instructs the AI assistant to guide architecture design toward reduced risk through redundancy, idempotent interfaces, independence, simplicity, and self-repair.
- **[architecture-risk-management](architecture-risk-management/SKILL.md)** — Guides the identification, quantification, and ongoing management of architecture risks using likelihood, severity, and a maintained risk matrix.
- **[architecture-scenario-based-modelling](architecture-scenario-based-modelling/SKILL.md)** — Guides the use of scenario-based modelling to capture multiple concrete domain stories, starting with the happy path and modeling important variations separately.
