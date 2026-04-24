---
name: architecture-reduce-risk
description: Instructs the AI assistant to guide architecture design toward reduced risk through redundancy, idempotent interfaces, independence, simplicity, and self-repair.
---

# Architecture Risk Reduction Instructions

When supporting architecture design, use this tool to recommend risk-reducing
principles such as safe redundancy, idempotent interfaces, component
independence, simplicity, and self-repair mechanisms.

---

## Purpose

This tool helps the AI assistant by:

- advising on how to reduce architectural risk without introducing harmful
  complexity,
- recommending safe redundancy strategies and idempotent interfaces,
- emphasizing true independence between components and infrastructure,
- promoting simplicity as a resilience strategy,
- suggesting self-repair mechanisms that improve availability without adding
  fragility.

---

## Expected outcome

As the AI, your response should help teams adopt an architecture that:

- reduces risk through carefully designed redundancy,
- uses idempotent interfaces to enable safe retries and recovery,
- ensures component independence from shared failure domains,
- keeps the system as simple as possible while meeting reliability goals,
- incorporates self-repair capabilities that do not create new risks.

---

## Instructions for the AI

1. **Explain safe redundancy**
   - Recommend redundancy across independent hardware, data centers, or services.
   - Warn against redundancy that increases hidden complexity or shared failure
     domains.
   - Suggest asynchronous or independent task execution where appropriate.

2. **Advocate idempotent interfaces**
   - Explain that idempotent operations can be retried safely without changing the
     outcome after the first successful call.
   - Give examples like setting a value versus incrementing a value.
   - Recommend designing APIs and commands to be idempotent whenever retries are
     expected.

3. **Stress true independence**
   - Advise verifying that redundant components do not share hidden infrastructure
     such as the same physical machine, rack, or power supply.
   - Recommend distributing failover targets across separate failure domains.
   - Highlight that perceived independence is not sufficient if underlying
     resources are shared.

4. **Promote simplicity**
   - Emphasize that complexity is the enemy of stability.
   - Recommend avoiding overly fine-grained microservice decompositions that add
     coordination overhead.
   - Encourage designs that keep individual services simple and minimize the
     total number of moving parts.

5. **Recommend self-repair mechanisms**
   - Suggest automated failover, retries with backoff, and rerouting around failed
     resources.
   - Recommend hot standby systems, resilient queues, and automated fault
     injection tests when appropriate.
   - Warn against self-repair systems that are so complex they create additional
     risk.

---

## AI decision guidance

When generating architecture risk reduction guidance, keep these principles in mind:

- **Balance redundancy with complexity:** recommend redundancy only when it
  genuinely reduces risk without creating a fragile architecture.
- **Prioritize idempotency:** design interfaces for safe retry semantics.
- **Validate independence:** ensure components are separated across real failure
  domains.
- **Keep it simple:** avoid recommending solutions that increase the system’s
  overall complexity unnecessarily.
- **Use self-repair wisely:** suggest automated recovery that is robust and
  maintainable, not brittle.

---

## Success criteria

A strong architecture risk reduction response should help teams achieve:

- **safer redundancy:** redundancy that improves availability without adding
  hidden failure modes,
- **retry-safe interfaces:** idempotent designs that simplify recovery,
- **true independence:** components separated by real infrastructure boundaries,
- **manageable architecture:** a low total complexity footprint,
- **resilient recovery:** self-repair mechanisms that increase availability
  without introducing new failure paths.

---

## Example prompts for the AI

- "Advise on how to make our service architecture more resilient without
  overcomplicating it."
- "Explain why idempotent APIs are important for retry and recovery strategies."
- "Describe how to build redundancy across truly independent failure domains."

---

## Related guidance

Use this tool alongside:

- architecture-building-with-failure-in-mind
- architecture-risk-management
- project-management-adaptive-project-management