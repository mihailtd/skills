# skills

[![Install via skills.sh](https://img.shields.io/badge/skills.sh-install-green)](https://skills.sh/mihailtd/skills)

A personal collection of [Agent Skills](https://agentskills.io/) for AI coding agents (GitHub Copilot, Claude Code, Cursor, Codex, and more).

## Installation

```bash
npx skills add mihailtd/skills
```

You will be prompted to select which skills to install and to which agents.

### Install a single skill

```bash
npx skills add mihailtd/skills --skill git-workflow
```

### Install all skills globally

```bash
npx skills add mihailtd/skills --all -g
```

## Available Skills

### [`git-workflow`](./skills/git-workflow/SKILL.md)

Best-practice guidelines for Git commits, branching, and pull requests. Follows Conventional Commits format and enforces clean history habits.

**Use when:**
- Writing commit messages
- Naming branches or opening pull requests
- Rebasing, merging, or resolving conflicts

### [`code-review`](./skills/code-review/SKILL.md)

Structured checklist for reviewing code changes for correctness, security, readability, testing, and performance.

**Use when:**
- Reviewing a pull request or diff
- Auditing code before merging
- Self-reviewing your own changes

### [`typescript-best-practices`](./skills/typescript-best-practices/SKILL.md)

TypeScript coding guidelines covering type safety, generics, null handling, imports, and `tsconfig.json` configuration.

**Use when:**
- Writing or reviewing TypeScript code
- Resolving type errors
- Designing shared types and interfaces

## Creating a New Skill

Scaffold a new skill with:

```bash
npx skills init my-skill
```

Each skill lives in its own directory under `skills/` and must contain a `SKILL.md` with valid YAML frontmatter (`name` and `description`).

## License

MIT
