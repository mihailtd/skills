---
name: python-clean-architecture-interface-segregation
description: Instructs the agent to satisfy the Interface Segregation Principle (clients shouldn't depend on operations they don't use) as an automatic consequence of functional dependency inversion — pass or bundle only the specific Callable functions a piece of code actually needs, never a class implementing unused methods or raising NotImplementedError for operations that don't apply. Extends python-clean-architecture-dependency-inversion with the ISP-specific framing.
---

# Python Clean Architecture: Interface Segregation, Functional-Lite

The Interface Segregation Principle (ISP) says clients shouldn't be forced
to depend on methods they don't use. The classic problem case is a fat
interface — one abstract class with many methods — that concrete classes
must fully implement even when some methods don't apply to them (often
answered with a `NotImplementedError` stub). In functional-lite, this
problem mostly **doesn't arise in the first place**: once dependencies are
expressed as individual `Callable` functions (see
python-clean-architecture-dependency-inversion) instead of interface
classes, a piece of code only ever receives the specific functions it
actually needs — there's no unused-method problem to solve, because
there's no bundled interface forcing unrelated operations together unless
you choose to bundle them.

---

## When to use this skill

Use this skill when you need to:

- design a dependency that different callers will use only part of (e.g.,
  a "player" that some callers use for audio-only operations, others for
  video-only operations),
- translate a fat-interface ISP example (one ABC with many
  `@abstractmethod`s, some raising `NotImplementedError` in certain
  subclasses) into this repo's style,
- decide how to bundle several related callables without accidentally
  recreating a fat interface,
- review code where a passed-in dependency object exposes operations the
  calling code never uses.

---

## Outcome

Produce dependency structures that:

- express each distinct operation as its own `Callable` type, never as one
  method among many on a shared interface,
- bundle only the operations a specific use actually needs into a
  `NamedTuple`/frozen `dataclass`, rather than one large bundle covering
  every possible operation across every possible caller,
- never contain a `NotImplementedError`/"this operation doesn't apply
  here" stub — if an operation doesn't apply to a given caller, that
  caller simply isn't given that callable at all,
- make it structurally impossible, not just conventionally discouraged,
  for a caller to depend on an operation it doesn't need.

---

## Instructions for the AI

1. **Reformulate a fat interface as independent `Callable` types**
   - Translate the book's `MultimediaPlayer` ABC (with `play_media`,
     `stop_media`, `display_lyrics`, `apply_video_filter` all bundled into
     one interface) into separate, independent `Callable` type aliases —
     there is no single fat interface type at all:
     ```python
     from typing import Callable

     PlayMedia = Callable[[str], None]
     StopMedia = Callable[[], None]
     DisplayLyrics = Callable[[str], None]
     ApplyVideoFilter = Callable[[str], None]
     ```
   - Each of these can be implemented by an independent plain function —
     `play_music`, `play_video`, `show_lyrics`, `apply_filter` — with no
     shared base type connecting them.

2. **Bundle only what a specific use case actually needs**
   - Instead of one `MultimediaPlayer` interface that every player type
     must fully implement, define a narrow `NamedTuple` per actual
     combination of operations a given caller needs:
     ```python
     from typing import NamedTuple

     class MusicPlayerOps(NamedTuple):
         play: PlayMedia
         stop: StopMedia
         show_lyrics: DisplayLyrics
         # No apply_video_filter field — it isn't in this bundle at all.

     class VideoPlayerOps(NamedTuple):
         play: PlayMedia
         stop: StopMedia
         apply_filter: ApplyVideoFilter
         # No show_lyrics field — it isn't in this bundle at all.
     ```
   - There is no `NotImplementedError` anywhere in this design — a music
     player bundle simply has no `apply_video_filter` field to call. The
     type system itself prevents the ISP violation, rather than relying on
     a convention (or a raised exception at runtime) to catch misuse.
   - Concrete function bundles are then just plain values:
     ```python
     music_ops = MusicPlayerOps(play=play_music, stop=stop_music, show_lyrics=show_lyrics)
     video_ops = VideoPlayerOps(play=play_video, stop=stop_video, apply_filter=apply_filter)
     ```

3. **Prefer individual parameters over a bundle when only one or two operations are needed**
   - Don't reach for a `NamedTuple` bundle by default — if a function only
     needs one or two callables, pass them as individual parameters
     directly (see python-clean-architecture-dependency-inversion step 3).
     Reserve bundling for cases where several related operations
     genuinely travel together across multiple call sites.
   - This keeps the "interface" a function actually depends on as small as
     the function's own signature, which is the most literal possible
     satisfaction of ISP — there's no bundle at all to be over-broad.

4. **Recognize ISP violations as a bundling mistake, not a missing-interface problem**
   - When reviewing code, treat a function or bundle that receives
     callables it never calls as the same underlying problem the book
     describes — the fix is to remove the unused callable(s) from the
     bundle/parameter list passed to that specific function, not to
     document that "this parameter isn't always used."
   - If a `NamedTuple` bundle is being passed around specifically because
     "everything might need any of these eventually," that's the fat-
     interface smell recurring in functional-lite form — split the bundle
     back into the narrower, actually-needed groupings per call site.

5. **Recognize this as reinforcing, not replacing, functional dependency inversion**
   - ISP isn't a separate technique to layer on top of
     python-clean-architecture-dependency-inversion — it's a property that
     falls out of doing dependency inversion correctly: pass exactly what's
     needed, nothing more. If dependency inversion is done with
     individually-scoped `Callable` parameters or narrowly-bundled
     `NamedTuple`s from the start, there's usually no separate "ISP pass"
     required later — this skill exists to make that connection explicit
     when translating ISP-specific reference material, not to introduce a
     new mechanism.

---

## Decision points and guidance

- **Does this bundle contain a callable that some callers never use?**
  Split the bundle into narrower, per-use-case bundles instead of adding a
  `None` default or a not-applicable stub.
- **Is a function receiving more callables than it actually calls?** Trim
  the parameter list or bundle down to exactly what's used.
- **Should this be a bundle, or individual parameters?** Default to
  individual parameters unless several operations genuinely travel
  together across multiple call sites.
- **Is there a `NotImplementedError`/"not supported here" stub anywhere in
  a dependency bundle?** That's the ISP violation resurfacing — remove the
  operation from that bundle entirely rather than stubbing it.

---

## Quality criteria

A strong ISP-satisfying dependency structure should ensure that:

- **no bundle contains an operation some of its consumers don't use**,
- **no `NotImplementedError`/not-applicable stub exists anywhere** in a
  dependency bundle — unused operations are simply absent, not stubbed,
- **bundles are scoped per actual use case**, not built as one
  general-purpose grab-bag of "everything a player might need,"
- **individual parameters are preferred over bundles** whenever only one
  or two operations are actually needed,
- **ISP is achieved as a natural consequence of correct dependency
  inversion**, not as a separate layer of interface design.

---

## Example prompts

- "This book example has a fat `MultimediaPlayer` interface with
  `NotImplementedError` stubs — reformulate it using narrow bundles of
  callables."
- "This function receives a big options bundle but only calls two of the
  five fields — help me trim that down."
- "Should I bundle these three callables into a NamedTuple, or just pass
  them as separate parameters?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-dependency-inversion
- python-clean-architecture-open-closed
- python-clean-architecture-single-responsibility
- python-clean-architecture-interface-adapters-boundary
- python-clean-architecture-controllers
- python-clean-architecture-drivers
