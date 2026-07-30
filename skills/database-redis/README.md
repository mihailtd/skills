# Database — Redis

Redis caching strategy guidance.

## Install

```bash
npx skills add mihailtd/skills/skills/database-redis --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/database-redis --skill database-redis-caching-strategy
```

## Skills (1)

- **[database-redis-caching-strategy](database-redis-caching-strategy/SKILL.md)** — Instructs the agent on advanced Redis caching strategies, including managing TTLs to prevent memory bloat, applying jitter to prevent cache stampedes, and using Lua scripting for atomic server-side caching.
