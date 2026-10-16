# API Review Checklist

Before completion, mark applicable checks passed, failed, or unverified with a reason; mark irrelevant checks not applicable.

## Wiring and Configuration

- Installed Express, TypeScript, schema, and middleware versions inspected; no automatic framework migration or snapshot patch pinning.
- App mounts and router paths compose exactly once; handlers, schema names, imports, repository methods, and Location headers agree.
- App is importable without starting a server or loading local development configuration. Runtime clients initialize after configuration.
- Proxy trust matches the deployed topology, CORS matches the project contract, public endpoints/preflight remain reachable as intended.
- Separate background workers do not execute HTTP authentication middleware.
- When Lambda Web Functions is selected, check the listen address, packaged entrypoint/dependencies, endpoint authentication, concurrency,
  and rollout settings using [lambda-web-functions.md](lambda-web-functions.md). Report deployed checks separately from local validation.

## Behavior and Boundaries

- Success, relevant not-found/conflict cases, and bodyless 204 responses preserve status, body, headers, and pagination contracts.
- Body/path/query failures and malformed JSON use the agreed format; trimmed/coerced/defaulted schema output reaches application logic.
- Authentication and group/scope/resource authorization prevent controller execution and writes when rejected. Trusted identity is verified.
- Unexpected async failures reach the four-parameter error handler; headers-sent failures delegate; sensitive diagnostics stay out of responses.
- Persistence stays in repositories, shared clients/context stay in composition, and services own workflows/integrations where needed.
- Cross-system cleanup retains failed work and is retryable when applicable; database deletion does not falsely imply physical object cleanup.

## Evidence and Handoff

- Relevant type checks, tests, and build pass. Real middleware is exercised through HTTP requests; test mocks do not bypass the behavior claimed.
- Tests use deterministic configuration, mocked external boundaries, disposable persistence where needed, and no unexpected outbound calls
  or unasserted warning/error logs. Local tests are distinguished from deployed authentication and live integration checks.
- Existing-pattern and small-project examples remain readable, with only required layers; standalone functions/callbacks use arrow functions.
- Report implemented routes, configuration/contract changes, checks and results, and limitations. Include useful snippets in the handoff.
- Update the active `stack-spec-workflow` Delivery Plan section or legacy active plan; do not add competing plans or approval gates.
