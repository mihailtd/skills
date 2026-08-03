---
name: architecture-network-api-versioning

description: Instructs the AI assistant that network API versioning differs from library versioning because multiple client versions and multiple server versions are simultaneously live in production, not sequentially upgraded — covers the customer-clarity questions to resolve before picking a strategy, the two dominant strategies (client-controlled, where the client requests an exact version and the server maintains every published version precisely, vs. server-controlled, where only major-version boundaries gate breaking changes and clients must tolerate unrecognized fields), and why a naive full-resource overwrite silently destroys data a calling client's version doesn't know about — the concrete case for PATCH-style partial updates.
---

# Network API Versioning Instructions

When supporting the design of a versioning strategy for an HTTP/network API —
not a library — use this tool. The core difference from library versioning
(**architecture-semantic-versioning**,
**architecture-library-api-evolution**): a library's consumers upgrade on
their own schedule, one build at a time, but a network API's old and new
clients, and old and new server instances, are simultaneously live in
production, continuously, for as long as multiple versions are supported.

---

## Purpose

This tool helps the AI assistant by:

- establishing that a network API's versioning problem is fundamentally
  about steady-state coexistence, not a brief transition — during any
  rolling deployment, old and new server instances run at the same time;
  for as long as an old client version is supported, it and newer clients
  read and write the same underlying data concurrently,
- working through the customer-clarity questions that should shape a
  versioning strategy before choosing one, rather than picking a strategy
  first and hoping it fits,
- explaining the two dominant strategies precisely — **client-controlled**
  (the client declares an exact version; the server maintains every
  supported version's precise behavior) and **server-controlled** (only a
  major version is negotiated; clients must tolerate response data they
  don't recognize) — and their real cost/precision tradeoff,
- connecting either strategy to the concrete risk of a naive full-resource
  update silently destroying data a particular client version doesn't know
  about, and making the case for PATCH-style partial updates as the fix.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- designs for old and new clients/servers coexisting continuously, not as a
  brief migration window — assumes multiple versions will be live in
  production simultaneously for as long as a support window lasts,
- answers the customer-clarity questions explicitly (typical calling
  context, communication channel for changes, evolution speed vs. consumer
  update speed, how long old versions stay supported, whether usage can be
  tracked per version) before committing to a strategy,
- picks client-controlled or server-controlled versioning deliberately based
  on precision-vs-cost tradeoffs that actually matter for the API in
  question, rather than by default or convention,
- for server-controlled versioning, treats "clients must tolerate
  unrecognized response fields" as a real requirement to design and
  document, not an assumption to hope holds,
- recognizes that a full-resource update endpoint risks silently deleting
  data whenever the calling client doesn't know about every field the
  resource has, and designs update semantics (PATCH-style, touching only
  explicitly-provided fields) that avoid this regardless of which
  versioning strategy is chosen.

---

## Instructions for the AI

1. **Design for continuous multi-version coexistence, not a transition window**
   - A rolling deployment means old and new server instances are both
     handling live traffic simultaneously — there's no instant, atomic
     moment where "the new version" replaces "the old version" the way a
     library consumer's rebuild does.
   - As long as an old client version is still supported, it's writing and
     reading the same underlying data as every newer client, concurrently,
     for the entire support window — potentially a long time, not a brief
     overlap. Design the data model and API contract with this steady-state
     coexistence in mind from the start, not as an edge case to patch in
     later.

2. **Resolve the customer-clarity questions before choosing a strategy**
   - **What's the typical calling context?** An API called almost
     exclusively by other backend services you control can support a more
     demanding versioning scheme than one called by long-lived IoT devices
     or client apps that may never be updated.
   - **Is there a reliable channel to warn consumers of upcoming changes?**
     Unlike an anonymous open-source library, a hosted API's operator often
     genuinely can contact known API consumers — use that if it's
     available; don't design as though every consumer is unreachable if
     they aren't.
   - **Does any part of the API need looser stability** for active
     collaboration with specific consumers, versus the rest of the surface
     being firmly stable?
   - **How fast does the API need to evolve, versus how fast can consumers
     realistically update?** A mismatch here is where a versioning strategy
     either becomes a bottleneck or accumulates unsupportable version
     sprawl.
   - **How long will old versions actually be supported**, and does that
     match what consumers expect or need? State this explicitly rather than
     leaving it implicit.
   - **Can usage be tracked per version, per client, per individual
     endpoint?** A hosted API usually can see its own traffic, unlike an
     open-source library — use that visibility to make real decisions about
     when a version is safe to retire, rather than guessing.

3. **Client-controlled versioning: precise, but the server carries the cost**
   - The client declares an exact version it understands (in a header,
     query parameter, or URL path — the exact mechanism matters less than
     the model). The server must know precisely which fields and behaviors
     belong to that version, and must not include anything from a later
     version in a request or response for an earlier one.
   - This gives strong precision — a client is never surprised by data or
     behavior it doesn't understand — but the server has to actually
     maintain every still-supported version's exact shape, typically via an
     internal, version-neutral representation with translation layers in
     and out for each supported version. That's real, ongoing implementation
     cost that scales with the number of versions kept alive simultaneously.
   - A practical version-string convention: major.minor without a patch
     component (patch-level differences are implementation details, not API
     surface, so they don't need client-facing versioning) — either a
     simple incrementing minor number or a date-based minor version
     (`1.20240615`) when at-a-glance recency is more useful than a short
     counter.

4. **Server-controlled versioning: cheaper, but clients must tolerate the unfamiliar**
   - Only a major version is negotiated. Within a major version, the API
     can only evolve in backward-compatible ways, and clients are expected
     to ignore any response data they don't recognize rather than treating
     it as an error — the API equivalent of Postel's Law ("be liberal in
     what you accept").
   - This is cheaper to implement than client-controlled versioning — far
     fewer distinct versions to maintain in parallel — but it pushes a real
     requirement onto every client: unrecognized fields must never cause a
     failure. Document this requirement explicitly for API consumers rather
     than assuming client implementations will naturally do the right
     thing.

5. **Either strategy needs PATCH-style updates to avoid silently destroying data**
   - A resource-update endpoint that accepts a "complete" representation of
     a resource and overwrites it wholesale is only complete from the
     calling client's point of view — a client on an older version, or one
     that simply doesn't know about a given field, has no way to include
     data it's never heard of. A server that blindly copies every field from
     the request onto the stored resource silently clears anything the
     client didn't (and couldn't) mention.
   - Client-controlled versioning has a partial defense here for free: the
     server already knows exactly which fields the client's declared
     version understands, and can preserve everything outside that set.
     Server-controlled versioning has no such signal and needs this
     handled explicitly — accept a request that only touches the fields it
     actually includes, and leave everything else on the stored resource
     untouched. See **python-fastapi-partial-updates** for the concrete
     Pydantic/FastAPI implementation of this pattern.
   - Decide this as part of the initial API design, not as a retrofit —
     switching an established full-replace endpoint to patch semantics
     later is itself a breaking change to the API's contract.

---

## AI decision guidance

When generating network API versioning guidance, keep these principles in
mind:

- **Multiple client and server versions coexist continuously in
  production** — design for steady-state coexistence, not a brief
  transition.
- **Resolve the customer-clarity questions explicitly before picking a
  strategy** — context, communication channel, evolution speed, support
  window, and trackability all shape which approach actually fits.
- **Client-controlled versioning trades server-side implementation cost for
  precision; server-controlled versioning trades that precision for lower
  cost and a "clients must tolerate the unfamiliar" requirement** — pick
  deliberately based on which cost the API can actually afford.
- **A full-resource update endpoint risks silently destroying data from any
  client that doesn't know about every field** — default to PATCH-style,
  only-touch-what's-provided semantics for anything where that data loss
  has a real cost.

---

## Success criteria

A strong response should ensure that it:

- **designs for continuous multi-version coexistence**, not a brief
  migration window,
- **works through the customer-clarity questions explicitly** before
  recommending a versioning strategy,
- **picks client-controlled or server-controlled versioning deliberately**,
  naming the specific cost/precision tradeoff that drove the choice,
- **states the "clients must tolerate unrecognized fields" requirement
  explicitly** when recommending server-controlled versioning,
- **recommends PATCH-style partial updates** wherever a full-resource
  overwrite risks silently destroying data a client doesn't know about.

---

## Example prompts for the AI

- "Should our API use client-controlled or server-controlled versioning?"
- "How long should we support an old API version before deprecating it?"
- "Is it safe for our update endpoint to just overwrite the whole resource
  with what the client sends?"
- "Our mobile clients update slowly — how should that affect our API
  versioning strategy?"

---

## Related guidance

Use this tool alongside:

- architecture-semantic-versioning / architecture-library-api-evolution —
  the library-versioning discipline this skill contrasts with; network APIs
  need a different model because of continuous multi-version coexistence.
- python-fastapi-partial-updates (package `python-fastapi`) — the concrete
  Pydantic/FastAPI implementation of the PATCH-style update pattern this
  skill's point 5 requires.
- architecture-idempotency-and-at-least-once-delivery — a PATCH-style
  partial update is naturally idempotent (safe to retry) in a way a
  poorly-designed full-replace update may not be, if replace semantics
  interact badly with concurrent writes.
- architecture-flexibility-complexity-tradeoffs — the general cost/precision
  tradeoff framing this skill's client-controlled-vs-server-controlled
  decision is a specific application of.
- architecture-data-storage-schema-evolution — the storage-layer counterpart
  to this skill's API-surface focus; the two timelines are deliberately
  independent, so a storage migration in progress doesn't force a breaking
  API change and vice versa.
