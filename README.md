# Stack Shifter Skills

> An AI agent skills library for building and sharing custom skills powered by the [open agent skills ecosystem](https://skills.sh/).

## Table of Contents

- [Stack Shifter Skills](#stack-shifter-skills)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Skills in This Repository](#skills-in-this-repository)
    - [`branch-risk-review`](#branch-risk-review)
    - [`spec-workflow`](#spec-workflow)
    - [`cdk-rest-api-dynamodb`](#cdk-rest-api-dynamodb)
    - [`cdk-rest-api-postgres`](#cdk-rest-api-postgres)
    - [`expressjs-rest-api`](#expressjs-rest-api)
  - [What Are Skills?](#what-are-skills)
  - [The Open Agent Skills Ecosystem](#the-open-agent-skills-ecosystem)
  - [Building Custom Skills](#building-custom-skills)
    - [Skill Structure](#skill-structure)
    - [Creating a Skill from Scratch](#creating-a-skill-from-scratch)
  - [Installing Skills](#installing-skills)
    - [Updating Installed Skills](#updating-installed-skills)
  - [Resources](#resources)

---

## Overview

This repository is a curated AI agent skills library maintained by Stack Shifter. Its purpose is to build, organize, and share reusable **custom skills** for AI agents — drawing from the [Agent Skills standard](https://agentskills.io/) and Anthropic's official [skills repository](https://github.com/anthropics/skills).

Skills let you extend AI agents with procedural knowledge: teach them your workflows, enforce your conventions, and automate repeatable tasks consistently across every session.

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

### `cdk-rest-api-dynamodb`

Builds CDK-based REST APIs with Lambda and DynamoDB using repo patterns first.

```bash
npx skills add stack-shifter/skills --skill cdk-rest-api-dynamodb
```

### `cdk-rest-api-postgres`

Builds CDK-based REST APIs with Lambda and Postgres via Drizzle using repo patterns first.

```bash
npx skills add stack-shifter/skills --skill cdk-rest-api-postgres
```

### `expressjs-rest-api`

Builds Express 5 REST APIs with TypeScript using repo patterns first.

```bash
npx skills add stack-shifter/skills --skill expressjs-rest-api
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

## The Open Agent Skills Ecosystem

[skills.sh](https://skills.sh/) is the open registry for discovering, publishing, and installing agent skills across the AI ecosystem. Skills published here are installable with a single command:

```bash
npx skills add <owner/repo>
```

To install the skills in this repository:

```bash
npx skills add stack-shifter/skills
```

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

## Examples

- Example usage 1
- Example usage 2

## Guidelines

- Guideline 1
- Guideline 2
```

**Required frontmatter fields:**

| Field         | Description                                       |
| ------------- | ------------------------------------------------- |
| `name`        | Unique identifier (lowercase, hyphens for spaces) |
| `description` | What the skill does and when to use it            |

### Creating a Skill from Scratch

**1. Name your skill**

Choose a lowercase, hyphen-separated name that describes the task, for example `release-checklist` or `aws-cost-audit`. This name becomes both the folder name and the `name` field in frontmatter.

**2. Create the skill folder and file**

```bash
mkdir my-skill
touch my-skill/SKILL.md
```

**3. Write the frontmatter**

Open `my-skill/SKILL.md` and add the required YAML frontmatter at the top:

```markdown
---
name: my-skill
description: A short, specific description of what this skill does and when the agent should use it.
---
```

The `description` field is especially important — agents use it to decide when to activate the skill, so be precise about the trigger context.

**4. Write the instructions**

Below the frontmatter, add a Markdown body with clear, imperative instructions. A recommended structure:

```markdown
# My Skill

Brief summary of what this skill does.

## When to Use

- Trigger condition 1
- Trigger condition 2

## Steps

1. First action the agent should take
2. Second action
3. Third action

## Examples

- `my-skill do X` — does X
- `my-skill do Y` — does Y

## Guidelines

- Keep outputs concise
- Always confirm before destructive actions
```

**5. Validate your skill**

Before committing, confirm:

- The folder contains a valid `SKILL.md`
- Frontmatter includes both `name` and `description`
- Any file paths or commands referenced in the skill match the actual repository layout

You can test by installing the repo locally:

```bash
npx skills add stack-shifter/skills
```

Then exercise the skill in a compatible agent to verify it activates and behaves as expected.

**6. Register the skill in this README**

Add a row to the [Skills in This Repository](#skills-in-this-repository) section with a short description and the per-skill install command:

```bash
npx skills add stack-shifter/skills --skill my-skill
```

Use the [official template](https://github.com/anthropics/skills/tree/main/template) from `anthropics/skills` as an additional reference.

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

**Claude Code:**

```bash
/plugin marketplace add stack-shifter/skills
```

### Updating Installed Skills

If you already installed this repo's skills and want the latest version after a skill changes, re-run the same `skills add` command.

**Refresh all skills from this repo:**

```bash
npx skills add stack-shifter/skills
```

**Refresh one specific skill:**

```bash
npx skills add stack-shifter/skills --skill <skill-name>
```

Examples:

```bash
npx skills add stack-shifter/skills --skill spec-workflow
npx skills add stack-shifter/skills --skill branch-risk-review
```

Use this after:

- editing a local skill in this repository
- pulling new changes from GitHub
- renaming or revising a skill and wanting the installed copy updated

---

## Resources

- [skills.sh](https://skills.sh/) — Open agent skills ecosystem and registry
- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skills library and reference implementations
- [Agent Skills Specification](https://agentskills.io/) — The open standard for agent skills
