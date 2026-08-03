---
name: architecture-standards-adoption
description: Instructs the AI assistant to guide the evaluation and adoption of technology standards (formal, de facto, or in-house) — verifying a standard actually fits the problem before adopting it, and layering a narrower in-house standard on top of a formal one when it permits more variation than the system should allow.
---

# Standards Adoption Instructions

When supporting architecture design, use this tool to help teams reason about
whether and how to adopt technology standards — distinguishing formal, de
facto, and in-house standards, verifying real suitability instead of adopting by
default, and layering narrower conventions on top of a standard that permits
more variation than the system wants.

---

## Purpose

This tool helps the AI assistant by:

- classifying a candidate standard as formal (backed by a standards body),
  de facto (broad adoption, no formal backing), or in-house (project- or
  org-specific convention),
- distinguishing when standards adoption is required by context versus
  merely suggested by it,
- pushing teams to verify a standard actually fits the specific problem
  before adopting it, rather than adopting by reputation or default,
- recommending "layering" a narrower in-house standard on top of a formal
  or de facto one when the underlying standard permits more variation than
  the system should allow,
- warning against both extremes: reflexive standards adoption without fit
  verification, and reinventing solved problems when a suitable standard
  already exists.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- correctly classifies any candidate standard by its tier (formal, de
  facto, in-house) and reasons about the implications of that tier,
- treats "standards exist for this" as a starting point for evaluation, not
  a substitute for verifying the standard fits this specific system,
  problem, and architectural style,
- captures the concrete benefits of adopting a well-fitting standard
  (reinforced roles/expectations, ecosystem familiarity, faster
  onboarding) as real architectural reasons, not just compliance
  checkboxes,
- layers a project- or org-specific standard on top of an adopted formal/de
  facto standard whenever that standard leaves more room for variation than
  the system should tolerate,
- is willing to recommend pushing back on a standard — even a popular
  de facto one — when it genuinely doesn't fit the problem at hand.

---

## Instructions for the AI

1. **Classify the candidate standard**
   - **Formal standard:** produced through a coordinating standards body
     (e.g., ISO) via multi-party cooperation; the strongest guarantee of
     documented, stable, broadly-implemented behavior.
   - **De facto standard:** broad adoption and multiple independent
     implementations, but no formal standards-body backing. Treat de facto
     status as real signal (it usually means the technology solved a
     genuine, common problem), but not as strong a guarantee as a formal
     standard — implementations can still diverge in undocumented ways.
   - **In-house standard:** a project's or organization's own convention —
     sometimes deliberately designed, sometimes just "how things were
     always done" that only gets recognized as a standard once someone
     deviates from it. Treat these as legitimate and worth documenting
     explicitly, not as a lesser category just because they're informal.

2. **Determine whether adoption is required or merely suggested**
   - **Required:** the product's context makes a standard non-negotiable
     (e.g., a product that exists specifically to expose an HTTP API must
     use HTTP). Recommend adopting it directly and move to fit
     verification.
   - **Suggested:** the context makes a standard attractive without
     mandating it (e.g., a network-accessible service that *could* use
     something other than HTTP, but where HTTP's ubiquity — client
     support, service support, developer familiarity — makes it a strong
     default). Recommend evaluating it seriously, but don't treat
     "suggested" as "automatic."

3. **Verify the standard actually fits before adopting it**
   - Explicitly check whether the standard's assumptions match the
     system's actual architectural style — e.g., HTTP assumes a
     client/server request-response style; confirm the system's real
     interaction pattern matches before adopting it as the reinforcing
     standard.
   - When a standard fits, point out the concrete payoff: it reinforces
     roles and expectations for components that already match its
     assumptions, and it lets the team draw on wide existing familiarity
     and tooling instead of building bespoke conventions and documentation.
   - When a standard does *not* fit — even a popular de facto one —
     recommend against forcing it, and be explicit about *why* it doesn't
     fit (which assumption breaks), not just that it "feels wrong."
     Treat this pushback as a normal, expected part of the job, not an
     exception to justify apologetically.

4. **Layer a narrower standard when the chosen one allows too much variation**
   - Recognize when an adopted standard is abstract enough to permit
     multiple valid ways of doing the same thing (e.g., HTTP allows both
     "POST to the resource's own URL" and "POST to the URL of its parent
     container" as valid resource-creation patterns).
   - When that variation would let different parts of the same system
     solve the same problem in different, needlessly inconsistent ways,
     recommend defining a narrower in-house standard on top of the formal/
     de facto one — pick one pattern, document it, and apply it
     consistently across the system.
   - Make clear that a layered in-house standard doesn't conflict with or
     replace the underlying standard — it's a compatible restriction of it,
     fully conformant to the broader standard while eliminating needless
     internal variation.

---

## AI decision guidance

When generating standards-adoption guidance, keep these principles in mind:

- **Classify before recommending:** know whether a candidate is formal, de
  facto, or in-house before reasoning about how much weight to give it.
- **Required versus suggested changes the conversation:** don't spend time
  "evaluating" a standard the context has already made mandatory; do spend
  real time evaluating one the context merely makes attractive.
- **Fit verification is not optional:** a standard's popularity or formal
  backing is not proof it fits this problem — always check the underlying
  assumptions against the system's actual architecture.
- **Pushing back is part of the job:** recommend against a standard when it
  genuinely doesn't fit, with a specific, concrete reason, even if it's
  widely used elsewhere.
- **Layer, don't refuse:** when a standard is directionally right but too
  permissive, the answer is usually a narrower in-house layer on top of it,
  not rejecting the standard altogether.

---

## Success criteria

A strong standards-adoption response should help teams achieve:

- **correct classification:** a clear read on whether a candidate is
  formal, de facto, or in-house, and what that implies for how much to
  trust it,
- **honest requirement assessment:** a clear distinction between "the
  context requires this" and "the context merely suggests this,"
- **verified fit:** a specific check of the standard's assumptions against
  the system's actual architectural style, not an assumption of fit,
- **captured reinforcement value:** concrete articulation of how adopting a
  well-fitting standard reinforces roles/expectations and leverages
  ecosystem familiarity,
- **eliminated needless variation:** an in-house layer defined wherever the
  chosen standard permits more variation than the system should tolerate.

---

## Example prompts for the AI

- "We're deciding whether to use HTTP for our internal service API — how do
  we determine if it actually fits, versus just being the popular choice?"
- "Our team keeps implementing REST resource creation two different ways —
  should we define our own convention on top of HTTP?"
- "Is this a case where we should push back on a widely-used standard
  because it doesn't fit our system?"

---

## Related guidance

Use this tool alongside:

- architecture-building-platforms
- architecture-risk-management
- architecture-reduce-risk
- architecture-capability-trajectory
