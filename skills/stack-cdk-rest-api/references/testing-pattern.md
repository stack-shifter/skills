# Isolated API Testing

Use when adding runtime, persistence, or CDK tests. Follow the existing runner and test layout; do not impose separate unit/integration
commands or filename conventions.

- Set deterministic test configuration before importing runtime composition. Never inherit application database URLs, AWS profiles, or
  developer credentials as test defaults.
- Mock AWS SDK and other external boundaries. Keep middleware under test real; handler composition tests may replace collaborators through
  intentional seams.
- For SQL behavior, use a disposable Testcontainers database and the project's committed migrations/required baseline fixture. Verify
  constraints, transactions, and concurrency invariants in the actual database engine rather than mocked query chains. Do not point
  automated tests at Compose or a deployed database.
- Block unexpected outbound requests, including swallowed failures. Permit only the explicit local transport needed by the container
  harness. Live authenticated browser/cloud checks are separate and require the appropriate authorization.
- Assert expected logs within the test that exercises them. Fail on unexpected warnings/errors instead of globally muting console output or
  hiding genuine failures.
- CDK template tests may disable bundling to focus on wiring. Separately synthesize with real bundling to verify entrypoints, runtime
  imports, permissions, and asset size; template-only tests prove neither execution nor deployed Cognito behavior.
- For regressions, demonstrate that the test fails without the fix and passes with it. Report tests, synthesis, and live checks separately;
  do not claim unrun checks.

## Deterministic Configuration and Fetch Guard Example

For Jest, register the first block through `setupFiles`, before importing application composition. Use non-routable test targets rather than
loading development environment files; replace required keys with the project's actual keys. Repository integration tests inject their
disposable container connection explicitly.

```ts
// test/setup-environment.ts
process.env.DATABASE_URL = 'postgresql://test:test@127.0.0.1:1/blocked';
process.env.CORS_ORIGIN = 'https://ui.example.invalid';
process.env.S3_BUCKET = 'example-test-bucket';
process.env.AWS_ACCESS_KEY_ID = 'test';
process.env.AWS_SECRET_ACCESS_KEY = 'test';
process.env.AWS_REGION = 'us-west-2';
process.env.AWS_EC2_METADATA_DISABLED = 'true';
delete process.env.AWS_SESSION_TOKEN;
delete process.env.AWS_PROFILE;
```

Register the next block through `setupFilesAfterEnv`. Tests that deliberately exercise fetch must replace it with a deterministic stub;
restoration returns to the blocked baseline. Recording attempts catches unexpected calls even when application code swallows the rejection.

```ts
// test/setup.ts
import { afterAll, afterEach, expect } from '@jest/globals';

const originalFetch = globalThis.fetch;
const unexpectedRequests: string[] = [];

const blockedFetch: typeof fetch = async (input) => {
	const url = input instanceof Request ? input.url : String(input);
	unexpectedRequests.push(url);

	throw new Error('Unexpected fetch in an isolated test.');
};

globalThis.fetch = blockedFetch;

afterEach(() => {
	const attempts = unexpectedRequests.splice(0);
	globalThis.fetch = blockedFetch;

	expect(attempts).toEqual([]);
});

afterAll(() => {
	globalThis.fetch = originalFetch;
});
```

This guard covers fetch only; AWS SDKs may use Node HTTP/HTTPS. Mock SDK boundaries and extend the existing harness to block those
transports before requests leave the process. Permit only required Docker/container transport. Do not copy a fetch-only guard and claim all
network traffic is isolated. Integrate restoration with the runner's existing cleanup hooks rather than adding competing setup mechanisms.

For expected error-path logs, keep suppression local and assert it:

```ts
const errorLog = jest.spyOn(console, 'error').mockImplementation(() => undefined);

try {
	await exerciseExpectedFailure();
	expect(errorLog).toHaveBeenCalledWith('Unable to load records.', expect.any(Error));
} finally {
	errorLog.mockRestore();
}
```

`exerciseExpectedFailure` represents the operation under test. Assert its observable failure result as well; a log assertion alone is not
behavior coverage. The suite's normal unexpected-log guard remains active for other tests.

## Error Contract Coverage

For an existing API, assert its established envelope and identifiers; do not migrate them to satisfy these examples. When Problem Details
is used, exercise these observable responses with real middleware and the existing runner:

- HTTP status matches body `status` when present; content type is `application/problem+json` (allow a charset parameter).
- Configured CORS is present on application errors, including parsing, validation, authorization, and unexpected failures.
- Missing body, malformed JSON, and body/query/path schema failures retain the agreed statuses and optional extensions.
- Field messages preserve multiple issues, nested/array paths such as `items.0.name`, root-level issues, and dictionary keys such as
  `__proto__`. Use safe fixture messages and verify rejected sensitive values are absent.
- Parsed/coerced/defaulted values reach controllers; JSON is parsed once. Failed validation does not execute business logic.
- Expected not-found/conflict reasons remain stable; fields/files appear only when relevant and access has been checked.
- Authentication/permission failures retain 401/403 or intentional 404 and required authentication headers. No unauthorized writes or
  sensitive resource disclosures occur.
- Unexpected exceptions produce safe generic 500 details; responses contain no stack traces, driver/SDK details, credentials, or sensitive
  rejected values. Scope expected-log spies to their test and assert redaction; do not globally mute logs.
- Success envelopes, Location headers, pagination, and bodyless 204 responses remain unchanged.

Use local deterministic fixtures and mocked external boundaries. Keep the parser/auth/error middleware under test real instead of replacing
it with a broad mock. Record deployed authorizer, Gateway/proxy, and live integration behavior as unverified unless separately checked.
