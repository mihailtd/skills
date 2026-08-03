---
name: python-dark-mode-light-mode-rollout
description: Instructs the agent on implementing the dark-mode/light-mode rollout technique in Python — running an old and new implementation side by side behind a RolloutMode-driven dispatcher, comparing outputs while returning the old result (dark) then the new one (light), running I/O-bound implementations concurrently via asyncio.gather to avoid doubling latency, and sampling the comparison rate for high-traffic paths. Use when swapping an implementation and needing production evidence that behavior hasn't changed before fully cutting over.
---

# Python Dark-Mode/Light-Mode Rollout Guidelines

You are an expert Python developer specializing in safely rolling out a
replacement implementation. When asked to swap out an algorithm, a data
access path, or any function whose behavior must provably stay unchanged,
use this pattern instead of a single, uncomparable cutover.

## 1. Model the Rollout State as an Enum, Not Two Booleans

Two independent booleans (`dark_mode`, `light_mode`) can be simultaneously
true or both false with no clear meaning. Use a single `Enum` so the state
is always unambiguous — this is the functional-lite, data-over-objects
default for this codebase (see the `python` skill).

```python
from enum import Enum, auto

class RolloutMode(Enum):
    OFF = auto()    # only the old implementation runs
    DARK = auto()   # both run; old result returned; discrepancies logged
    LIGHT = auto()  # both run; new result returned; discrepancies logged
    ON = auto()     # only the new implementation runs
```

## 2. Implement Old and New Side by Side, Behind a Dispatcher

Keep both implementations as ordinary functions and route through a single
dispatcher that reads the current `RolloutMode` (from config/feature-flag
state — see `python-pydantic-configuration`).

```python
import asyncio
import logging

logger = logging.getLogger(__name__)

async def calculate_score_old(user_id: str) -> float:
    ...  # existing implementation

async def calculate_score_new(user_id: str) -> float:
    ...  # replacement implementation

async def calculate_score(user_id: str, mode: RolloutMode) -> float:
    if mode is RolloutMode.OFF:
        return await calculate_score_old(user_id)
    if mode is RolloutMode.ON:
        return await calculate_score_new(user_id)

    # DARK or LIGHT: run both, compare, return based on mode
    old_result, new_result = await asyncio.gather(
        calculate_score_old(user_id),
        calculate_score_new(user_id),
    )
    if old_result != new_result:
        logger.warning(
            "rollout discrepancy",
            extra={"function": "calculate_score", "user_id": user_id,
                   "old_result": old_result, "new_result": new_result},
        )
    return old_result if mode is RolloutMode.DARK else new_result
```

**Run both implementations concurrently with `asyncio.gather`, not
sequential `await` calls, whenever they're I/O-bound.** This is a
Python-specific improvement on the naive version of this pattern: awaiting
each implementation one after another doubles wall-clock latency, but
`asyncio.gather` runs them concurrently, so the added latency is closer to
`max(old, new)` than `old + new`. This doesn't reduce the doubled load on
downstream resources (the database still sees two queries) — only the
caller-facing latency cost.

Use structured logging (see `python-configuration-observability`) for the
discrepancy log, not a bare print/string — a discrepancy needs to be
searchable and aggregable across many requests to be useful evidence.

## 3. Stage the Rollout: `OFF` → `DARK` → `LIGHT` → `ON`, Never Skipping a Stage

- **`DARK`**: callers never see a behavior change — the old result is
  always returned. Fix every discrepancy that surfaces before moving on.
- **`LIGHT`**: callers now see the new result, but comparison continues —
  this also catches a concurrent, unrelated change landing in
  `calculate_score_old` that isn't mirrored in the new implementation,
  which `DARK` mode alone wouldn't have caught.
- **`ON`**: only after a stable, discrepancy-free soak period at full
  `LIGHT` exposure. This is the trigger to start the cleanup in step 5.

Widen exposure gradually within `DARK` and `LIGHT` (a rollout percentage,
a subset of environments) rather than flipping the whole system from `OFF`
straight to full `DARK`/`LIGHT` at once.

## 4. Sample the Comparison for High-Traffic Paths

Dual execution and comparison cost real resources at volume — extra
database/network load, and a logging system that can itself become a
bottleneck if every request logs a comparison. For a high-traffic
candidate, sample the comparison instead of running it on every call:

```python
import random

SAMPLE_RATE = 0.05  # start low; ramp up as discrepancies quiet down

async def calculate_score(user_id: str, mode: RolloutMode) -> float:
    if mode is RolloutMode.OFF:
        return await calculate_score_old(user_id)
    if mode is RolloutMode.ON:
        return await calculate_score_new(user_id)

    if random.random() >= SAMPLE_RATE:
        # Skip comparison this call; just serve the mode's primary result.
        return (
            await calculate_score_old(user_id)
            if mode is RolloutMode.DARK
            else await calculate_score_new(user_id)
        )

    old_result, new_result = await asyncio.gather(
        calculate_score_old(user_id), calculate_score_new(user_id)
    )
    if old_result != new_result:
        logger.warning("rollout discrepancy", extra={...})
    return old_result if mode is RolloutMode.DARK else new_result
```

Increase `SAMPLE_RATE` as discrepancies quiet down, working toward 100% (or
a stable, low, accepted rate) before advancing rollout stages.

## 5. Clean Up Once Stable — This Is Part of the Rollout, Not Optional

Once `ON` and stable, delete `calculate_score_old`, the dispatcher, the
`RolloutMode` plumbing for this candidate, and any now-dead feature-flag
config — `calculate_score_new`'s logic should end up living directly under
the original `calculate_score` name. Leaving the dispatcher and dual
implementations in place after cutover reintroduces the exact stale
transitional-artifact problem a completed refactor should avoid.

## Related guidance

- **architecture-verified-behavior-rollout** (package `architecture`) —
  the conceptual technique and cost/benefit tradeoffs this skill
  implements concretely in Python.
- **architecture-refactoring-plan-structure** (package `architecture`) —
  the mandatory cleanup milestone this skill's step 5 feeds into.
- **python-configuration-observability** — structured logging conventions
  for the discrepancy log.
- **python-pydantic-configuration** — where the `RolloutMode`/rollout
  percentage configuration should live.
