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

### Creating a Skill

1. Create a new folder under `skills/` named after your skill
2. Add a `SKILL.md` file using the structure above
3. Write clear, specific instructions the agent should follow
4. Include usage examples and guidelines for best results

Use the [official template](https://github.com/anthropics/skills/tree/main/template) from `anthropics/skills` as a starting point.

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

<!-- Add new skills below this line -->

---

## Resources

- [skills.sh](https://skills.sh/) — Open agent skills ecosystem and registry
- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skills library and reference implementations
- [Agent Skills Specification](https://agentskills.io/) — The open standard for agent skills
- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills) — Claude support documentation
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills) — Step-by-step skill creation guide
- [Skills API Quickstart](https://docs.claude.com/en/api/skills-guide#creating-a-skill) — Using skills via the Claude API
