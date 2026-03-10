# Repository Guidelines

## Coding Philosophy

Prefer code that is:

- Agents must prioritize simple, readable, and maintainable code over clever or highly abstract solutions.
- Prefer explicit control flow (if statements, for loops, early returns) and clear variable names rather than functional chaining or complex patterns.
- Some duplication is acceptable if it improves readability and keeps logic easy to follow.
- Avoid introducing abstractions, utilities, or libraries unless the same logic appears at least three times.
- When multiple solutions exist, always choose the most boring, straightforward implementation that a mid-level to senior developer could understand quickly.
- Include reasonable, concise comments to explain intent and non-obvious logic, but avoid excessive comments that clutter or restate obvious code.
- Do not generalize code for hypothetical future use; implement only what is required for the current feature.
- Prefer small functions with a single responsibility and avoid deeply nested logic.
- Follow existing patterns in the repository instead of introducing new architectural styles.
- Do not introduce complex design patterns (factories, builders, dependency injection layers, etc.) unless required.

## Project Structure & Module Organization

This repository is a lightweight library of agent skills. Today, the root contains project documentation in `README.md` and `LICENSE`; contributed skills should be added as their own top-level directories, each with a `SKILL.md` entry point. Use a layout like `my-skill/SKILL.md`, and keep any supporting assets local to that skill directory so the skill stays portable.

## Build, Test, and Development Commands

There is no compiled build step in this repository. Use these commands during development:

- `npx skills add stack-shifter/skills` installs this repo through the skills ecosystem for local validation.
- `npx skills add <owner/repo>` verifies compatibility with the standard install flow described in `README.md`.
- `git status` checks that your contribution only includes the intended skill files and docs updates.

If you add scripts later, document them in `README.md` and keep command examples copy-pasteable.

## Coding Style & Naming Conventions

Write `SKILL.md` files in clear Markdown with YAML frontmatter. Use lowercase, hyphen-separated skill names in frontmatter and directory names, for example `release-checklist` or `aws-cost-audit`. Keep prose direct, imperative, and task-oriented. Prefer short sections such as `## When to Use`, `## Steps`, and `## Examples`. Use 2-space indentation in YAML and fenced code blocks for shell examples.

## Testing Guidelines

There is no automated test suite yet, so test contributions by installing the repository and exercising the skill in an agent that supports the Agent Skills format. At minimum, confirm that:

- the skill folder contains a valid `SKILL.md`
- frontmatter includes `name` and `description`
- examples and referenced paths match the repository layout

## Commit & Pull Request Guidelines

Git history is minimal, so use short commit subjects prefixed by the change type, such as `feat: add terraform-audit skill`, `fix: correct install command`, `refactor: simplify skill instructions`, or `bug: resolve broken example path`. Keep each subject concise and scoped to one change. Pull requests should include a short summary, the reason for the skill or edit, and sample usage when behavior is not obvious. Include screenshots only when the change affects rendered documentation.
