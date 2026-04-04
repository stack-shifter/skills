# Stack Shifter Skills

> An AI agent skills library for building and sharing custom skills powered by the [open agent skills ecosystem](https://skills.sh/).

## Table of Contents

- [Stack Shifter Skills](#stack-shifter-skills)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Skills in This Repository](#skills-in-this-repository)
    - [`branch-risk-review`](#branch-risk-review)
    - [`spec-workflow`](#spec-workflow)
    - [`dynamodb-design`](#dynamodb-design)
    - [`cdk-rest-api`](#cdk-rest-api)
    - [`expressjs-rest-api`](#expressjs-rest-api)
    - [`prototype`](#prototype)
    - [`repo-templatizer`](#repo-templatizer)
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

### `branch-risk-review`

Reviews the current branch against `main` for breaking changes and merge risk.

```bash
npx skills add stack-shifter/skills --skill branch-risk-review
```

### `spec-workflow`

Runs a locked-spec, phased-plan workflow for feature delivery.

```bash
npx skills add stack-shifter/skills --skill spec-workflow
```

### `dynamodb-design`

Designs DynamoDB schemas from access patterns first, including single-table vs multi-table choices, key design, GSIs, relationships, pagination, uniqueness, TTL, and schema reference documentation.

```bash
npx skills add stack-shifter/skills --skill dynamodb-design
```

### `cdk-rest-api`

Builds CDK-based REST APIs with Lambda using repo patterns first, including route composition, auth, middleware, runtime wiring, schedules, and repository-backed persistence boundaries.

```bash
npx skills add stack-shifter/skills --skill cdk-rest-api
```

### `expressjs-rest-api`

Builds Express 5 REST APIs with TypeScript using repo patterns first.

```bash
npx skills add stack-shifter/skills --skill expressjs-rest-api
```

### `prototype`

Builds frontend-only UI prototypes from requirements, generating self-contained HTML screen or flow files plus a companion product spec document.

```bash
npx skills add stack-shifter/skills --skill prototype
```

### `repo-templatizer`

Converts an existing repository into a reusable GitHub template while preserving reusable structure and infrastructure patterns.

```bash
npx skills add stack-shifter/skills --skill repo-templatizer
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
└── my-skill/
    └── SKILL.md
```

The `SKILL.md` file uses YAML frontmatter followed by Markdown instructions:

```markdown
---
name: my-skill-name
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

Choose a lowercase, hyphen-separated name that describes the task, for example `release-checklist` or `aws-cost-audit`. This name becomes both the folder name and the `name` field in frontmatter.

**2. Scaffold the skill**

Use the `skills` CLI to generate the template automatically:

```bash
npx skills init my-skill
```

This creates `my-skill/SKILL.md` pre-populated with the structure shown in [Skill Structure](#skill-structure).

**3. Fill in the frontmatter**

The `description` field is especially important — agents use it to decide when to activate the skill, so be precise about the trigger context.

**4. Write the instructions**

Replace the placeholder body with clear, imperative instructions following the structure in [Skill Structure](#skill-structure).

**5. Validate and register**

Before committing, confirm the folder contains a valid `SKILL.md` with both `name` and `description`, and that any referenced file paths match the actual repository layout. Then install and exercise the skill in a compatible agent (see [Installing Skills](#installing-skills)).

Finally, add an entry to the [Skills in This Repository](#skills-in-this-repository) section with a short description and the per-skill install command:

```bash
npx skills add stack-shifter/skills --skill my-skill
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
