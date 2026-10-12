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
