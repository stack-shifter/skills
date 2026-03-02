# Stack Shifter Skills

> An AI agent skills library for building and sharing custom skills powered by the [open agent skills ecosystem](https://skills.sh/).

## Table of Contents

- [Stack Shifter Skills](#stack-shifter-skills)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [What Are Skills?](#what-are-skills)
  - [The Open Agent Skills Ecosystem](#the-open-agent-skills-ecosystem)
  - [Building Custom Skills](#building-custom-skills)
    - [Skill Structure](#skill-structure)
    - [Creating a Skill](#creating-a-skill)
  - [Installing Skills](#installing-skills)
  - [Supported Agents](#supported-agents)
  - [Skills in This Repository](#skills-in-this-repository)
  - [Resources](#resources)

---

## Overview

This repository is a curated AI agent skills library maintained by Stack Shifter. Its purpose is to build, organize, and share reusable **custom skills** for AI agents — drawing from the [Agent Skills standard](https://agentskills.io/) and Anthropic's official [skills repository](https://github.com/anthropics/skills).

Skills let you extend AI agents with procedural knowledge: teach them your workflows, enforce your conventions, and automate repeatable tasks consistently across every session.

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

---

## Supported Agents

Skills built to the [Agent Skills standard](https://agentskills.io/) are compatible with a wide range of AI agents, including:

Claude Code · GitHub Copilot · Cursor · Windsurf · Cline · Codex · Gemini · Goose · Roo · and [many more](https://skills.sh/)

---

## Skills in This Repository

A brief description of every skill available in this library, along with its individual install command.
### `branch-risk-review`

Compares the current branch to `main` to produce a structured, risk-focused review that surfaces breaking changes, regression risk, security regressions, data/migration risks, and operational concerns. Use when reviewing a PR, preparing a release, or answering "what changed vs main?".

```bash
npx skills add stack-shifter/skills --skill branch-risk-review
```

### `approval-spec-workflow`

Enforces an approval-gated, two-phase workflow: produce a detailed spec first, then implement only after explicit approval. Trigger by prefixing any prompt with `Spec:`. Keeps architecture and implementation decisions explicit and reviewable before any code is written.

```bash
npx skills add stack-shifter/skills --skill approval-spec-workflow
```

### `cdk-rest-api-dynamodb`

Designs and implements REST API endpoints on AWS using the target repository's CDK constructs first, with Lambda handlers and DynamoDB as the default datastore. Use whenever you need to add or modify API Gateway routes, Lambda handlers, Cognito auth, request models, or DynamoDB-backed CRUD operations.

```bash
npx skills add stack-shifter/skills --skill cdk-rest-api-dynamodb
```

---

## Resources

- [skills.sh](https://skills.sh/) — Open agent skills ecosystem and registry
- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skills library and reference implementations
- [Agent Skills Specification](https://agentskills.io/) — The open standard for agent skills
- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills) — Claude support documentation
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills) — Step-by-step skill creation guide
- [Skills API Quickstart](https://docs.claude.com/en/api/skills-guide#creating-a-skill) — Using skills via the Claude API
