---
name: python-clean-architecture-legacy-assessment
description: Instructs the agent to evaluate an existing (non-Clean-Architecture) Python codebase and plan its transformation — a targeted preliminary analysis (dependency mapping, framework penetration, dispersed business rules) translated into business-impact terms for stakeholders, Event Storming for collaborative domain discovery (mapping directly to entities/use-cases/domain-rules by sticky-note color), and a staged roadmap (Foundation, Interface, Integration, Optimization) with an explicit work-organization strategy. This is the planning phase — see python-clean-architecture-incremental-migration for safely executing the plan against a running system.
---

# Python Clean Architecture: Assessing and Planning a Legacy Migration

Transforming an existing, tangled codebase toward Clean Architecture starts
with assessment and planning, not code. This skill covers evaluating a
legacy system through Clean Architecture's lens, translating findings into
terms that secure stakeholder support, using Event Storming to discover
domain boundaries collaboratively, and turning the result into a staged,
resourced roadmap. Once a stage is planned, the actual entity/use-case/
repository reformulation follows the same functional-lite patterns already
established throughout this package (python-clean-architecture-domain-modeling,
python-clean-architecture-dependency-inversion, etc.) — this skill is about
deciding what to do and in what order, not re-teaching those patterns.

---

## When to use this skill

Use this skill when you need to:

- evaluate an existing Python system for Clean Architecture violations
  before proposing a transformation,
- translate technical architectural problems into terms that win
  stakeholder and resourcing support,
- run collaborative domain discovery (Event Storming) to find natural
  architectural boundaries in a system that never had them,
- turn assessment findings into a staged, prioritized transformation
  roadmap,
- decide how to organize the transformation work alongside ongoing feature
  development.

---

## Outcome

Produce a transformation plan that:

- is grounded in a targeted (not exhaustive) preliminary analysis —
  enough concrete violations to make a credible, business-relevant case,
  not a complete architectural inventory,
- has real stakeholder buy-in, backed by baseline metrics that will later
  demonstrate the transformation's value,
- reflects domain boundaries discovered collaboratively (via Event
  Storming) rather than boundaries a single engineer inferred alone from
  reading code,
- is organized into clear stages (Foundation, Interface, Integration,
  Optimization) with an explicit, deliberate choice of how the work fits
  around ongoing feature delivery.

---

## Instructions for the AI

1. **Run a targeted preliminary analysis, not an exhaustive audit**
   - Look for a small number of concrete, illustrative violations across
     four areas, enough to make the case credibly without exhaustively
     documenting the whole system:
     - **Architectural inventory:** the major components and how they
       interact — a baseline sketch, not a complete map.
     - **Dependency mapping:** the most problematic circular dependencies
       and framework coupling — the handful that actually hurt, not every
       relationship.
     - **Framework penetration:** concrete places framework code has
       visibly invaded business logic (an ORM model with business rules
       attached, a route handler containing pricing logic).
     - **Domain logic dispersion:** a few clear examples of a single
       business rule fragmented across multiple files/modules.
   - Resist the urge to diagram everything at this stage — deeper
     understanding comes from the collaborative domain analysis (step 3),
     not from a solo exhaustive audit up front.

2. **Translate findings into business-impact terms before engaging stakeholders**
   - Reframe each technical finding in terms non-technical stakeholders
     feel directly: "changing how pricing works currently requires edits
     in seven files across three modules" lands harder than "pricing logic
     is duplicated." Connect findings to time-to-market, bug rates, and
     the system's ability to respond to changing requirements.
   - Identify the right stakeholders for the scale of transformation
     planned: engineering (technical constraints), product (business
     priorities and value), operations (deployment/reliability), and end
     users (pain points) — a small refactor needs only the immediate team;
     a system-wide overhaul needs leadership engagement.
   - Establish baseline metrics before starting, across four categories —
     maintenance (bug-fix time, feature lead time), quality (defect rate,
     test coverage), team effectiveness (onboarding time, deploy
     frequency), and business outcomes (customer satisfaction, feature
     adoption) — these justify the initial investment, validate progress
     as work proceeds, and help define what "done" means (sustainable
     improvement, not architectural perfection).

3. **Use Event Storming for collaborative domain discovery**
   - Recommend Event Storming specifically among the available
     collaborative discovery techniques (domain storytelling, context
     mapping) when the goal is finding Clean Architecture boundaries in an
     existing system — its color-coded sticky-note vocabulary maps
     directly onto this package's layers, making architectural boundaries
     tangible to non-technical participants:
     - **Orange (domain events)** — things that happened (`Order
       Placed`) — map to **entities** at the domain's core.
     - **Blue (commands)** — actions taken (`Process Payment`) — map to
       **use cases** in the application layer.
     - **Purple (business rules)** — constraints that hold regardless of
       delivery mechanism — map to **domain-level invariants** (see
       python-clean-architecture-entity-invariants).
   - Use the same entity/value-object/aggregate vocabulary from
     python-clean-architecture-domain-modeling during the session — this
     is the same domain-modeling discipline, now applied to discover what
     an existing system *should* have separated but didn't.
   - Treat natural fault lines the session surfaces (e.g., "Order
     processing and Payment processing change for different reasons and
     at different rates") as strong signals for where module/aggregate
     boundaries belong — this is the Single Responsibility "reason to
     change" heuristic (see python-clean-architecture-single-responsibility)
     surfacing at the whole-system level.

4. **Organize the transformation into four stages**
   - **Foundation:** establish core domain concepts — entities, value
     objects, and repository/service `Callable` types — alongside the
     existing implementation, without touching the running system yet.
     This is exactly python-clean-architecture-domain-modeling and
     python-clean-architecture-dependency-inversion applied to the
     concepts Event Storming surfaced.
   - **Interface:** build the adapters that bridge the new domain core to
     what already exists — repository implementations working against the
     *existing* database schema, use-case functions, and controller
     functions (see python-clean-architecture-drivers,
     python-clean-architecture-use-cases,
     python-clean-architecture-controllers) — still without cutting the
     running system over.
   - **Integration:** migrate real functionality onto the new path,
     feature by feature or domain by domain — see
     python-clean-architecture-incremental-migration for the concrete
     patterns (Strangler Fig, feature flags, Shadow Mode) that make this
     safe.
   - **Optimization:** refine based on real-world experience once the
     migration has landed — performance tuning in drivers, expanded test
     coverage, better error handling — treating the target architecture as
     something reached through continuous refinement, not a single pass.
   - Track the baseline metrics from step 2 through every stage — this is
     what turns "we think this is better" into a demonstrable result, and
     helps recognize when the transformation has reached a good-enough
     stopping point rather than chasing architectural perfection
     indefinitely.

5. **Choose how the work fits around ongoing feature delivery**
   - **Dedicated transformation iterations:** sprint cycles allocated
     exclusively to architectural work — good for components needing
     significant change completable within one or two iterations, at the
     cost of delaying feature delivery during those cycles.
   - **Parallel transformation tracks:** a dedicated team works
     architecture while others continue features — maintains delivery
     velocity, needs real coordination to avoid conflicts, fits larger
     systems where transformation spans multiple quarters.
   - **Opportunity-based transformation:** refactor a component toward
     Clean Architecture whenever a feature change already touches it —
     lowest isolated risk, but progress depends on feature priorities and
     can leave the system unevenly transformed.
   - Recommend combining these deliberately rather than picking one
     globally: critical, high-change components often warrant dedicated
     effort; rarely-touched areas can wait for opportunity-based
     transformation. Make this choice explicit per component, not assumed.

---

## Decision points and guidance

- **Is the preliminary analysis turning into an exhaustive audit?** Pull
  back — a handful of credible, concrete violations is enough to start
  stakeholder conversations; deeper understanding comes from collaborative
  discovery, not a solo deep-dive.
- **Have stakeholders and baseline metrics been established before
  planning specific changes?** If not, secure buy-in and a measurement
  baseline first — this is what justifies the investment and later proves
  it worked.
- **Were domain boundaries discovered collaboratively, or inferred
  unilaterally from code?** Prefer Event Storming or an equivalent
  collaborative technique — boundaries that emerge from stakeholder
  discussion tend to hold up better than ones inferred from reading
  legacy code alone.
- **Is a single work-organization approach being applied uniformly?**
  Check whether different components actually warrant different
  approaches (dedicated vs. parallel vs. opportunity-based) rather than
  defaulting to one for the whole system.

---

## Quality criteria

A strong legacy assessment and plan should ensure that:

- **the preliminary analysis is targeted and credible**, not exhaustive,
- **findings are translated into business-impact language** before
  stakeholder engagement, with baseline metrics established up front,
- **domain boundaries come from collaborative discovery**, using Event
  Storming's direct mapping to entities/use-cases/domain-rules,
- **the roadmap has explicit stages** (Foundation, Interface, Integration,
  Optimization) with metrics tracked throughout,
- **the work-organization approach is chosen deliberately per component**,
  not applied as a single blanket strategy.

---

## Example prompts

- "Help me put together a preliminary architectural analysis of this
  legacy Flask app to make the case for a Clean Architecture
  transformation."
- "Walk me through running an Event Storming session to find domain
  boundaries in this order-processing system."
- "Help me build a staged roadmap for transforming this system, and decide
  which components need dedicated effort versus opportunity-based
  refactoring."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-incremental-migration
- python-clean-architecture-domain-modeling
- python-clean-architecture-single-responsibility
- architecture-domain-storytelling
- architecture-incremental-delivery
