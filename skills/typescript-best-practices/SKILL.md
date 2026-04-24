---
name: typescript-best-practices
description: TypeScript coding guidelines and patterns. Use when writing, reviewing, or refactoring TypeScript code, creating types or interfaces, configuring tsconfig, or resolving type errors.
license: MIT
metadata:
  author: mihailtd
  version: "1.0.0"
---

# TypeScript Best Practices

Guidelines for writing safe, maintainable, and idiomatic TypeScript.

## When to Use

- Writing new TypeScript files or modules
- Reviewing TypeScript code for type safety
- Resolving `tsc` or ESLint type errors
- Designing shared types and interfaces
- Configuring `tsconfig.json`

## Core Rules

### Type Safety
- Enable `strict: true` in `tsconfig.json` — never disable it per-file
- Avoid `any`; prefer `unknown` when the type is genuinely unknown and narrow with type guards
- Never use non-null assertions (`!`) without a comment explaining why it is safe
- Use `as const` for literal object/array values instead of widening to `string` or `number`

### Types vs. Interfaces
- Prefer `interface` for object shapes that may be extended
- Use `type` for unions, intersections, mapped types, and aliases
- Do not use `I` prefix for interfaces (e.g., `User`, not `IUser`)

### Functions
- Always annotate return types on exported functions
- Use function overloads for polymorphic behaviour instead of `any` parameters
- Prefer named exports over default exports for better refactoring support

### Enums
- Prefer string literal unions over `enum` to avoid runtime overhead and bundler compatibility issues
  ```ts
  // Prefer
  type Direction = "north" | "south" | "east" | "west";
  // Over
  enum Direction { North, South, East, West }
  ```

### Generics
- Constrain generics (`<T extends object>`) to avoid accidental widening
- Name generic parameters meaningfully when there are multiple: `<TInput, TOutput>` not `<T, U>`

### Null Handling
- Use optional chaining (`?.`) and nullish coalescing (`??`) instead of manual null checks
- Return `undefined` from functions that may not produce a value; avoid returning `null` unless the API requires it

### Imports
- Use type-only imports where possible: `import type { Foo } from './foo'`
- Avoid barrel re-exports in large codebases — they hurt tree-shaking and increase build time

## tsconfig.json Baseline

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "moduleResolution": "bundler",
    "esModuleInterop": true
  }
}
```

## Common Failure Modes

- **Casting with `as`**: Only use when you have more type information than the compiler; document why
- **Type assertions on external data**: Always validate/parse data from APIs, forms, or storage with a schema library (e.g., Zod, Valibot) rather than asserting types
- **Overly wide return types**: Returning `object` or `any[]` when a precise type is available
