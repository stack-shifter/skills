# Stack Shifter Skills

> An AI agent skills library for building and sharing custom skills powered by the [open agent skills ecosystem](https://skills.sh/).

## Table of Contents

- [Stack Shifter Skills](#stack-shifter-skills)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Skills in This Repository](#skills-in-this-repository)
    - [`stack-branch-risk-review`](#stack-branch-risk-review)
    - [`stack-spec-workflow`](#stack-spec-workflow)
    - [`stack-dynamodb-design`](#stack-dynamodb-design)
    - [`stack-storm-rest-api`](#stack-storm-rest-api)
    - [`stack-storm-websocket-api`](#stack-storm-websocket-api)
    - [`stack-expressjs-rest-api`](#stack-expressjs-rest-api)
    - [`stack-prototype`](#stack-prototype)
    - [`stack-repo-templatizer`](#stack-repo-templatizer)
  - [What Are Skills?](#what-are-skills)
  - [Building Custom Skills](#building-custom-skills)
    - [Skill Structure](#skill-structure)
    - [Creating a Skill from Scratch](#creating-a-skill-from-scratch)
  - [Installing Skills](#installing-skills)
  - [Resources](#resources)

---

## Overview

This repository is a curated AI agent skills library maintained by Stack Shifter. Its purpose is to build, organize, and share reusable **custom skills** for AI agents — drawing from the [Agent Skills standard](https://agentskills.io/) and Anthropic's official [skills repository](https://github.com/anthropics/skills).

Skills let you extend AI agents with procedural knowledge: teach them your workflows, enforce your conventions, and automate repeatable tasks consistently across every session. Skills in this repo are installable via [skills.sh](https://skills.sh/) with `npx skills add stack-shifter/skills`.

---

## Skills in This Repository

A brief description of every skill available in this library, along with its individual install command.

### `stack-branch-risk-review`

Reviews the current branch against `main` for breaking changes and merge risk.

```bash
npx skills add stack-shifter/skills --skill stack-branch-risk-review
```

### `stack-spec-workflow`

Runs a locked-spec workflow for feature delivery, with clarification-first drafting and phase-by-phase implementation check-ins.

```bash
npx skills add stack-shifter/skills --skill stack-spec-workflow
```

### `stack-dynamodb-design`

Designs DynamoDB schemas from access patterns first, including single-table vs multi-table choices, key design, GSIs, relationships, pagination, uniqueness, TTL, and schema reference documentation.

```bash
npx skills add stack-shifter/skills --skill stack-dynamodb-design
```

### `stack-storm-rest-api`

Builds CDK-based REST APIs with Lambda using repo patterns first, including route composition, auth, middleware, runtime wiring, schedules, and repository-backed persistence boundaries.

```bash
npx skills add stack-shifter/skills --skill stack-storm-rest-api
```

### `stack-storm-websocket-api`

Builds WebSocket APIs with Storm, including connection authentication, message middleware, controllers, connection repositories, and callback delivery.

```bash
npx skills add stack-shifter/skills --skill stack-storm-websocket-api
```

Example: “Use stack-storm-websocket-api to add an authenticated updates route with connection tracking and notifications to authorized recipients.”

### `stack-expressjs-rest-api`

Builds Express 5 REST APIs with TypeScript using repo patterns first.

```bash
npx skills add stack-shifter/skills --skill stack-expressjs-rest-api
```

### `stack-prototype`

Builds frontend-only UI prototypes from requirements, generating self-contained HTML screen or flow files plus a companion product spec document.

```bash
npx skills add stack-shifter/skills --skill stack-prototype
```

### `stack-repo-templatizer`

Converts an existing repository into a reusable GitHub template while preserving reusable structure and infrastructure patterns.

```bash
npx skills add stack-shifter/skills --skill stack-repo-templatizer
```

---

## What Are Skills?

Skills are **reusable, self-contained capabilities for AI agents**. Each skill is a folder containing a `SKILL.md` file with structured instructions and metadata that an agent loads dynamically to improve performance on a specialized task.

Skills can teach an agent how to:

- Create documents following your brand guidelines
- Analyze data using your organization's specific workflows
- Automate personal or team tasks in a repeatable way
- Follow coding standards and architectural patterns

---

## Building Custom Skills

This repository uses the [anthropics/skills](https://github.com/anthropics/skills) repository as both a reference and foundation. Anthropic's skills system defines the standard for how skills are structured and consumed by Claude and other compatible agents.

### Skill Structure

Each skill lives in its own folder and requires a single `SKILL.md` file:

```
skills/
└── stack-my-skill/
    └── SKILL.md
```

The `SKILL.md` file uses YAML frontmatter followed by Markdown instructions:

```markdown
---
name: stack-my-skill
description: A clear description of what this skill does and when to use it
---

# My Skill Name

Instructions that the agent will follow when this skill is active.

## When to Use

- Trigger condition 1
- Trigger condition 2

## Steps

1. First action the agent should take
2. Second action
3. Third action

## Guidelines

- Keep outputs concise
- Always confirm before destructive actions
```

**Required frontmatter fields:**

| Field         | Description                                       |
| ------------- | ------------------------------------------------- |
| `name`        | Unique identifier (lowercase, hyphens for spaces) |
| `description` | What the skill does and when to use it            |

### Creating a Skill from Scratch

**1. Name your skill**

Choose a lowercase, hyphen-separated name prefixed with `stack-`, for example `stack-release-checklist` or `stack-aws-cost-audit`. This name becomes both the folder name and the `name` field in frontmatter.

**2. Scaffold the skill**

Use the `skills` CLI to generate the template automatically:

```bash
npx skills init stack-my-skill
```

This creates `stack-my-skill/SKILL.md` pre-populated with the structure shown in [Skill Structure](#skill-structure).

**3. Fill in the frontmatter**

The `description` field is especially important — agents use it to decide when to activate the skill, so be precise about the trigger context.

**4. Write the instructions**

Replace the placeholder body with clear, imperative instructions following the structure in [Skill Structure](#skill-structure).

**5. Validate and register**

Before committing, confirm the folder contains a valid `SKILL.md` with both `name` and `description`, and that any referenced file paths match the actual repository layout. Then install and exercise the skill in a compatible agent (see [Installing Skills](#installing-skills)).

Finally, add an entry to the [Skills in This Repository](#skills-in-this-repository) section with a short description and the per-skill install command:

```bash
npx skills add stack-shifter/skills --skill stack-my-skill
```

---

## Installing Skills

Skills from this repository can be used directly in Claude Code, Claude.ai, and via the Claude API.

**Install all skills:**

```bash
npx skills add stack-shifter/skills
```

**Install a single skill:**

```bash
npx skills add stack-shifter/skills --skill <skill-name>
```

Re-running either command also updates previously installed skills to the latest version.

---

## Resources

- [skills.sh](https://skills.sh/) — Open agent skills ecosystem and registry
- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skills library and reference implementations
- [Agent Skills Specification](https://agentskills.io/) — The open standard for agent skills
