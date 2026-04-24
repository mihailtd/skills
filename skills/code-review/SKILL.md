---
name: code-review
description: Review code for quality, security, and maintainability. Use when asked to review code, audit a PR, check for bugs, identify security issues, or suggest improvements to existing code.
license: MIT
metadata:
  author: mihailtd
  version: "1.0.0"
---

# Code Review

A structured checklist for reviewing code changes for quality, security, correctness, and maintainability.

## When to Use

- Reviewing a pull request or diff
- Auditing a file or module before merging
- Self-reviewing your own changes before opening a PR
- Identifying security vulnerabilities or bugs

## Review Checklist

### Correctness
- [ ] Logic matches the stated requirements
- [ ] Edge cases are handled (empty input, null/undefined, boundary values)
- [ ] Error paths return or throw appropriately — no silent failures
- [ ] Async code handles rejections and race conditions

### Security
- [ ] No secrets, tokens, or credentials committed
- [ ] User input is validated and sanitized before use
- [ ] SQL/NoSQL queries use parameterized statements (no string concatenation)
- [ ] Authentication and authorization are enforced on every sensitive route
- [ ] Dependencies are up-to-date and free of known CVEs

### Readability & Maintainability
- [ ] Function and variable names are self-documenting
- [ ] Functions do one thing and stay ≤ 40 lines when reasonable
- [ ] Complex logic has a brief explanatory comment (the *why*, not the *what*)
- [ ] No dead code, commented-out blocks, or TODO comments without issue references

### Testing
- [ ] New logic has corresponding unit or integration tests
- [ ] Tests cover happy path, error path, and meaningful edge cases
- [ ] Tests are independent and deterministic — no order dependencies

### Performance
- [ ] No N+1 queries or unnecessary network calls in loops
- [ ] Heavy computations are not run on every render/request when they can be cached
- [ ] Large data structures are not serialized unnecessarily

## How to Review

1. Read the PR description and linked issue to understand *intent*.
2. Walk the diff top-to-bottom, applying the checklist above.
3. Categorize feedback:
   - **Blocker**: Must fix before merge (bugs, security issues)
   - **Suggestion**: Recommended improvement
   - **Nit**: Minor style preference — prefix with "nit:"
4. Be specific: quote the line, explain the problem, and suggest a fix.
5. Approve only when all blockers are resolved.
