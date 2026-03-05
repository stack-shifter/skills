---
name: repo-templatizer
description: Prepares an existing repository to become a reusable GitHub template. Use this whenever the user wants to genericize a repo, scrub project-specific domain logic, preserve reusable structure, turn a real project into starter boilerplate, or make a codebase safe to publish as a template. This includes requests to keep `src/` and `test/` layouts, retain reusable services, rewrite the README into placeholders, or preserve CDK `constructs/` code while removing app-specific implementation. Also use this when the user wants to check for or remove branding, org references, company names, badges, account IDs, or project-specific identifiers without restructuring the codebase — this activates Branding Audit Mode.
---

# Repo Templatizer

Use this skill to convert a concrete repository into a reusable GitHub template without flattening the architecture that makes the project valuable.

The goal is not to erase the repo into an empty shell. The goal is to preserve reusable structure and framework wiring while replacing project-specific logic with minimal, obvious starter examples.

## Outcome

Produce a repository that:

- keeps the established folder layout, especially under `src/` and `test/`
- removes or genericizes project-specific domain logic, naming, sample data, and business flows
- preserves reusable utilities, framework setup, infrastructure primitives, and generic service abstractions
- replaces removed implementation with small boilerplate examples or `.gitkeep` where no safe example belongs
- rewrites `README.md` into a generic template README that still reflects the repo's real setup
- adds a dedicated customization guide that tells template users exactly what to rename, replace, or configure

## Portability Rule

This skill must work across repositories. Do not assume a specific framework, language, package manager, or cloud provider.

Source of truth order:

1. The target repository's current structure and conventions
2. These instructions
3. Small generated examples only where needed

If the repository already has a clean reusable pattern, keep it. Do not replace a good abstraction with a more generic but worse one.

## Mode Selection

Before starting, identify which mode applies based on the user's request.

- **Full Templatization Mode** (default): Run the full skill — Discovery Pass through Output Expectations. Use when the user wants to genericize the whole repo, scrub domain logic, or produce a reusable template.
- **Branding Audit Mode**: Run only the branding pass described below. Use when the user explicitly asks to check for or remove branding, org references, or project-specific identifiers without touching source code structure or implementation.

If the user's request is ambiguous, default to Full Templatization Mode and note what was done.

---

## Branding Audit Mode

Use this mode when the user wants branding removed or reviewed without restructuring the codebase.

**Scope:** Surface identifiers only. Do not touch `src/`, `test/`, domain logic, folder structure, or boilerplate examples.

### What to Scan

Inspect the following locations for branding:

- `README.md` and any other root-level docs (`CONTRIBUTING.md`, `CHANGELOG.md`, etc.)
- Package manifests: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `build.gradle`, `pom.xml`, etc.
- CI/CD configs: `.github/workflows/`, `.circleci/`, `Jenkinsfile`, `bitbucket-pipelines.yml`, etc.
- Environment example files: `.env.example`, `.env.sample`, `config/example.*`
- Infrastructure configs: any file referencing account IDs, org names, hosted URLs, or service names
- License files: license holder name or org name
- Docker and container configs: image names, registry URLs, org prefixes
- `package.json` `name`, `repository`, `homepage`, `bugs`, and `author` fields
- Git config or GitHub Actions that reference the old repo owner or org

### What Counts as Branding

Flag and replace:

- Company or organization names (full name, abbreviation, or slug)
- Product or project names that are not generic
- Personal names in author fields, license headers, or commit templates
- Hosted URLs or domains tied to the old org or product
- Account IDs, org slugs, tenant IDs, or deployment identifiers embedded in config
- Badges (shields.io, etc.) referencing the old repo owner, package registry name, or CI status
- Repository URLs, npm/PyPI/crates package names, or container registry paths
- Email addresses tied to the old org

### What to Leave Alone

Do not modify:

- `src/`, `test/`, or any implementation files
- Generic dependency names, framework names, or tool versions
- Placeholder values already using a bracket or `TODO` convention
- CI steps or scripts that describe build behavior, not identity

### Replacement Strategy

Replace identified branding with safe placeholders:

| Category | Placeholder |
|---|---|
| Project / product name | `[PROJECT_NAME]` |
| Organization / company | `[ORG_NAME]` |
| Repository URL | `[REPO_URL]` |
| Package name | `[PACKAGE_NAME]` |
| Deployed service URL | `[SERVICE_URL]` |
| Account or tenant ID | `[ACCOUNT_ID]` |
| Author name | `[AUTHOR_NAME]` |
| Author email | `[AUTHOR_EMAIL]` |
| Container image | `[CONTAINER_IMAGE]` |
| Registry path | `[REGISTRY_PATH]` |

Use consistent placeholders throughout — if `[PROJECT_NAME]` appears in the README it should appear in `package.json` too.

### Output for Branding Audit Mode

Report:

- Every file where branding was found and replaced, with the specific field or line
- Any branding that was flagged but not replaced (explain why)
- A list of all unique placeholders introduced so the template consumer knows what to search-replace after cloning

Do not produce a `TEMPLATE_CUSTOMIZATION.md` unless the user asks for it. Do not rewrite the README beyond replacing branding tokens.

---

## Discovery Pass

Start with a focused repository scan before proposing changes.

Inspect:

- root docs such as `README.md`, contribution docs, and template-related files
- package manifests and lock files
- `src/`, `test/`, and equivalent runtime directories
- infra folders such as `cdk/`, `infra/`, `lib/`, `constructs/`, or `stacks/`
- service, utility, shared, common, config, and dependency-wiring modules
- environment examples and CI files if they document setup assumptions

Classify files into these buckets:

1. `Keep as reusable foundation`
2. `Genericize but preserve shape`
3. `Replace with boilerplate example`
4. `Remove and leave .gitkeep`

State the mode after discovery:

- `Structure-preserving mode`: the repo already has reusable architecture worth keeping
- `Heavy-scrub mode`: much of the app is domain-bound and should be reduced to starter examples

## Core Preservation Rules

Always preserve:

- the top-level repository shape unless it is obviously broken
- the folder skeleton under `src/` and `test/`
- framework bootstrapping, build configuration, linting, formatting, and test harness wiring when they are reusable
- generic services, adapters, clients, shared middleware, helpers, and utilities that are not tightly bound to the old domain
- reusable infrastructure code and deployment primitives

Do not preserve:

- company- or product-specific business rules
- domain entities, workflows, handlers, controllers, and test fixtures that only make sense for the old project
- proprietary names, URLs, IDs, accounts, secrets, ARNs, and environment defaults
- README copy that describes the old product as if it still exists

## `src/` and `test/` Rules

Keep the structure that exists under `src/` and `test/` even when major logic is removed.

When reducing files:

- preserve meaningful directories
- keep representative entrypoints and one or two small example modules where that helps users understand the intended architecture
- prefer `.gitkeep` only when a directory should remain but any sample implementation would mislead users

Good candidates for starter examples:

- a simple health-check route
- an example service with a trivial pure function
- a sample repository interface with a no-op or in-memory implementation
- one minimal test per preserved testing style

Bad candidates for starter examples:

- fake domain workflows that still look like real business logic
- large seeded fixtures
- copied production data models with renamed nouns

## Reusable vs Domain-Specific Logic

Keep services when they are genuinely reusable, such as:

- logging
- configuration loading
- auth middleware or auth helpers that are provider-agnostic enough to reuse
- persistence wrappers, repository base classes, or data clients
- HTTP clients, retry helpers, caching helpers, feature flag wrappers, or event utilities
- validation helpers and generic schema utilities

Genericize or remove services when they encode domain behavior, such as:

- order, billing, claims, inventory, identity, tenant, or product workflows
- domain-specific orchestration
- reports and exports tied to old business entities
- tests that assert old domain scenarios

When unsure, keep the reusable shell and replace the internals with neutral examples.

## Boilerplate Replacement Strategy

When replacing domain code, prefer the smallest example that proves how the architecture is supposed to be used.

Good replacement pattern:

1. Preserve file names or rename to neutral names if the old names are obviously domain-specific
2. Keep interfaces, DI shape, and composition patterns if they are reusable
3. Replace the body with simple examples such as `ExampleService`, `SampleController`, `HealthController`, or `ExampleConstruct`
4. Update tests to match the neutral example

If a file has no useful neutral example:

- delete project-specific contents
- preserve the directory
- add `.gitkeep` if the directory would otherwise disappear from git

## README Conversion

Rewrite `README.md` into a generic starter document derived from the original repository.

Preserve:

- the real framework and tooling stack
- real install, build, test, and run commands when still valid
- the real folder structure if it helps users orient themselves
- any reusable setup steps

Replace with placeholders:

- project name
- business/domain description
- environment-specific identifiers
- screenshots, URLs, API names, and branded references

The README should read like a template consumers can adopt immediately. Use placeholders such as:

- `[PROJECT_NAME]`
- `[PROJECT_DESCRIPTION]`
- `[PRIMARY_DOMAIN_ENTITY]`
- `[AWS_ACCOUNT_ID]`
- `[SERVICE_NAME]`

Include short notes where users are expected to rename or replace starter examples.

Keep `README.md` concise. It should introduce the template, explain how to run it, and point users to a dedicated customization guide for repo-specific follow-up.

## Customization Guide

Create a dedicated file such as `TEMPLATE_CUSTOMIZATION.md` or `CUSTOMIZE.md`.

Prefer `TEMPLATE_CUSTOMIZATION.md` unless the target repository already uses another naming convention for onboarding or post-generation tasks.

This file should be the exhaustive handoff for template consumers. Include:

- placeholders that must be replaced, such as project name, package name, service name, org name, account IDs, URLs, and domain entities
- directories or files that intentionally still contain starter example code
- reusable services or constructs that were preserved and the places where users are expected to wire in their own domain logic
- environment variables, secrets, deployment settings, and infrastructure names that must be customized
- tests, fixtures, sample data, and screenshots that should be rewritten or removed
- a checklist of first-follow-up tasks after creating a repo from the template

When useful, organize the file by path so users can work through the repo systematically.

`README.md` should link to this file near the top.

## GitHub Template Readiness

Prepare the repo for GitHub template consumption.

Check for and genericize:

- template-unfriendly project names in package manifests and docs
- stale issue references, personal links, and internal company references
- example env files that still contain real values
- CI defaults that assume the old org, account, or deployment target
- badges or release metadata tied to the old repository

Prefer safe placeholders over deleting useful guidance.

## CDK Rule

When the repository is a CDK project, preserve the `constructs/` folder and all custom constructs.

Apply these rules:

- keep reusable custom constructs intact
- keep construct APIs and patterns that express reusable infrastructure design
- remove or genericize only stack composition, resource naming, route wiring, seed data, or business-domain integrations that are specific to the old project
- if a construct is narrowly domain-specific but structurally useful, rename and neutralize it rather than deleting the construct pattern

Do not collapse a CDK repo into raw stack files if the reusable value lives in the constructs layer.

## Working Rules

1. Read the relevant runtime, test, manifest, and infrastructure files before editing.
2. Preserve architecture first, then genericize content.
3. Keep the minimum viable example code needed to teach future users how to extend the template.
4. Prefer neutral names like `Example`, `Sample`, `Health`, `Template`, or resource-role names over recycled business nouns.
5. Do not leave dead imports, dead scripts, or stale docs after scrubbing domain logic.
6. Keep tests aligned with the new boilerplate so the template still demonstrates how testing works.
7. Summarize what was preserved, what was genericized, and what was intentionally left for template users to customize.
8. Generate a dedicated customization guide and ensure `README.md` points to it.

## Output Expectations

When you finish work in a target repository, report:

- what structure was preserved
- which domain areas were replaced with examples or `.gitkeep`
- what reusable services or constructs were intentionally retained
- how `README.md` was converted for template use
- where the customization guide was added and what categories of follow-up it covers
