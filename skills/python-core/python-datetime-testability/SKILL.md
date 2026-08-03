---
name: python-datetime-testability

description: Instructs the agent on writing testable date/time code — injecting a Clock (a Callable[[], datetime] type alias, functional-lite, never a class hierarchy) instead of calling datetime.now() directly, always passing an explicit IANA zone instead of relying on system-local defaults, avoiding locale-sensitive parsing/formatting for machine-facing text, and splitting a public entry point that gathers ambient context from a pure, easily-testable inner function that takes everything as explicit parameters.
---

# Python Date/Time Testability Guidelines

You are an expert Python developer specializing in testable date/time code. When asked to write logic that depends on the current time, a time zone, or locale, adhere to the following rules.

## 1. Inject the Current Time — Never Call `datetime.now()` Directly Inside Logic

A function that calls `datetime.now()` internally can't be tested against a specific instant — you can't reliably make the system clock read exactly one minute before a target, so entire classes of edge-case tests (boundary conditions, off-by-one-second bugs) become impossible to write deterministically.

Inject a clock as a `Callable[[], datetime]` parameter instead of a class hierarchy — this repo's functional-lite style has no need for the abstract-`Clock`/`SystemClock`-singleton/`FakeClock` trio a class-based language reaches for; a plain callable does the same job:

```python
from collections.abc import Callable
from datetime import datetime, timedelta, timezone

Clock = Callable[[], datetime]

def is_within_one_minute_of_target(target: datetime, clock: Clock) -> bool:
    now = clock()
    return abs(now - target) <= timedelta(minutes=1)
```

In production, bind the clock to the real system time:

```python
from functools import partial

production_clock: Clock = partial(datetime.now, timezone.utc)
```

In tests, a clock is just a function that returns a fixed value — no fake class needed:

```python
import pytest

@pytest.mark.parametrize("offset_seconds", [-61, 61])
def test_outside_target_interval(offset_seconds: int) -> None:
    target = datetime(2024, 1, 1, tzinfo=timezone.utc)
    fixed_clock: Clock = lambda: target + timedelta(seconds=offset_seconds)
    assert not is_within_one_minute_of_target(target, fixed_clock)

@pytest.mark.parametrize("offset_seconds", [-60, -30, 60])
def test_within_target_interval(offset_seconds: int) -> None:
    target = datetime(2024, 1, 1, tzinfo=timezone.utc)
    fixed_clock: Clock = lambda: target + timedelta(seconds=offset_seconds)
    assert is_within_one_minute_of_target(target, fixed_clock)
```

For code you don't control the construction of (a third-party call site, or a quick script), `monkeypatch`ing `datetime.now` (see **python-testing-mocking**) or a library like `freezegun`/`time-machine` is a reasonable fallback — but prefer clock injection for anything you're writing yourself, since it makes the dependency visible in the function signature instead of hidden behind a module-level patch.

## 2. Always Pass an Explicit IANA Zone — Never Rely on the System-Local Default

Several `datetime`/`zoneinfo` operations silently fall back to the system's local time zone when no zone is given — `datetime.now()` (no args, also naive — see **python-datetime-modeling**), `datetime.astimezone()` (no args converts to system-local), and `date.today()`. Relying on these defaults means the code's behavior depends on where it happens to be deployed, which is exactly the kind of hidden assumption that breaks the moment a container runs somewhere new:

```python
from zoneinfo import ZoneInfo

# risky: behavior silently depends on the machine's configured local zone
local_now = datetime.now().astimezone()

# correct: explicit, works identically on any machine
delivery_zone = ZoneInfo("Australia/Sydney")
now_at_delivery = clock().astimezone(delivery_zone)
```

If a function needs a time zone to do its job, take it as an explicit parameter — don't give it a default that silently reaches for the system zone, even if the system zone happens to be correct today. A caller three or four calls removed from the function that actually needs the zone should not have to know that fact exists; making the parameter explicit and required is what prevents that surprise.

## 3. Split a Thin Public Entry Point From a Pure, Explicitly-Parameterized Inner Function

When a public method naturally has access to ambient context (an order's shipping details, a request's delivery address), gather that context in a thin public function and delegate immediately to an inner function that takes everything explicitly — the inner one is what you actually test:

```python
def get_final_returns_date(order_item: OrderItem, clock: Clock) -> date:
    shipping_time = order_item.shipping_details.warehouse_exit_time
    delivery_zone = order_item.order.delivery_address.time_zone
    return _compute_final_returns_date(shipping_time, delivery_zone)

def _compute_final_returns_date(shipping_time: datetime, delivery_zone: ZoneInfo) -> date:
    # the actual calendar-arithmetic logic, fully testable with two plain arguments
    ...
```

This is the same functional-core/imperative-shell split used throughout this repo's Clean Architecture skills (see **python-clean-architecture-dependency-inversion**) — the inner function is trivial to call from a test with exactly the instant and zone a scenario needs, without constructing a full `OrderItem`/`Order` object graph just to exercise the date logic.

## 4. Avoid Locale-Sensitive Parsing and Formatting for Machine-Facing Text

`strftime`/`strptime` format codes like `%c`, `%x`, and bare month-name codes are locale-sensitive — the same code produces different text (or fails to parse text it previously produced) depending on the process's configured locale. For anything machine-facing (logs, APIs, storage), use an explicit, locale-independent format — `datetime.isoformat()` for output, and parse with an explicit format string or `datetime.fromisoformat()` rather than a locale-dependent parser. Reserve locale-aware formatting (via a library like `babel`) specifically for text actually displayed to an end user in their own locale — not for anything another program will read.

## Related guidance

- **python-datetime-modeling** — the instant/civil-date/civil-datetime/duration/period concept model this skill's testability patterns operate on.
- **python-datetime-requirements-clarity** — resolving *what* a date/time feature should do before writing the testable code this skill covers *how* to structure.
- **python-testing-mocking** — `monkeypatch`-based alternatives for code where clock injection isn't practical.
- **python-clean-architecture-dependency-inversion** — the general `Callable`-type-alias dependency-injection pattern this skill's clock injection is a specific application of.
- **architecture-third-party-testability-evaluation** (package `architecture`) — evaluating whether a *third-party* library exposes this same kind of injectable-clock seam before depending on it.
