---
name: git-workflow
description: Guide Git commit, branch, and PR workflows. Use when asked to commit changes, create a branch, write a commit message, open a pull request, or manage version control tasks.
license: MIT
metadata:
  author: mihailtd
  version: "1.0.0"
---

# Git Workflow

Best-practice guidelines for Git commits, branching, and pull requests.

## When to Use

- Writing commit messages
- Creating or naming branches
- Opening pull requests
- Rebasing, merging, or resolving conflicts
- Reviewing or explaining Git history

## Commit Messages

Follow the [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<optional scope>): <short summary>

[optional body]

[optional footer(s)]
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`, `revert`

Rules:
- Summary line ≤ 50 characters (hard limit 72), imperative mood ("add feature" not "added feature")
- Reference issue numbers in the footer: `Closes #123`
- One logical change per commit — avoid "WIP" or "misc fixes" commits

## Branch Naming

Pattern: `<type>/<short-description>`

Examples:
- `feat/add-oauth-login`
- `fix/memory-leak-dashboard`
- `chore/upgrade-dependencies`
- `docs/update-readme`

Rules:
- Lowercase, hyphens only (no underscores or spaces)
- Keep descriptions short and descriptive (≤ 40 chars)
- Delete branches after merging

## Pull Requests

A good PR:
1. Has a clear title following Conventional Commits format
2. Has a description explaining *what* changed and *why*
3. References related issues (`Closes #N`)
4. Is small and focused — one concern per PR
5. Has all CI checks passing before requesting review

## Common Failure Modes

- **Huge commits**: Split into logical units; use `git add -p` for partial staging
- **Force-pushing shared branches**: Never force-push `main`/`master` or shared feature branches
- **Merge conflicts from stale branches**: Rebase often (`git fetch && git rebase origin/main`)
- **Missing context in messages**: Future readers cannot read your mind — explain the *why*
