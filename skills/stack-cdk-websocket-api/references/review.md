# WebSocket Review

Mark applicable checks passed, failed, or unverified with a reason. Static inspection alone does not verify runtime or deployed behavior.

## Infrastructure

- Selected library or project construct exports and signatures match examples.
- Registry values resolve to functions in the correct entry modules without initializing runtime dependencies during synthesis.
- Connect authorizer configuration is separate from route defaults. Only connect has Gateway authentication; message authorization is enforced in application code.
- Client URL and management endpoint are used correctly, including custom domains and stages.
- Defaults, stage settings, route responses, and resource grants synthesize as intended. Only callback functions receive management access.

## Behavior Scenarios

1. Valid handshake saves verified identity; missing/invalid credentials and failed persistence do not produce successful connections.
2. A valid updates message reaches its controller with parsed values. Invalid JSON, empty text, and unknown actions return the agreed response without publishing.
3. Missing/expired sender state or denied permission prevents recipient lookup and callback writes. Recipients stay within current receive authorization.
4. Delivery skips the sender for this example, filters expired recipients, handles all pages, and reports partial results. Gone connections are removed; other failures do not delete records or stop every later recipient.
5. Disconnect cleanup succeeds for an absent record. Expiry and stale cleanup still work if no disconnect event arrives.
6. Acknowledgments with route responses enabled and explicit callbacks reach the expected clients independently. No callback is attempted before a handshake completes.

## Validation and Handoff

- Check frontmatter, Markdown, portable paths, examples, and install instructions when editing the skill.
- Type-check examples against actual Storm and Middy dependencies, and run focused tests with mocked storage, token verification, permission service, and callback clients.
- Synthesize affected CDK stacks and inspect auth, stage, response, and IAM configuration when dependencies are available. Do not deploy as a validation shortcut.
- For already authorized live testing, use two test clients to verify connection, update, acknowledgment, recipient isolation, and disconnect behavior. Never present mock tests as live delivery verification.
- Report completed routes, contract choices, tests, and remaining limitations. Record progress in the active delivery plan when using stack-spec-workflow.

## Infrastructure Selection Checks

- With Storm installed, prefer its supported constructs and verify their signatures.
- Without Storm, extend existing project constructs or synthesize project-local constructs without adding a Storm dependency.
- Explicit user preferences win; do not migrate existing APIs automatically.
- Exercise both infrastructure paths with the same application behavior and permission checks.

- Without Storm, stack examples call local construct route methods; native Lambda, API, stage, and integration wiring stays inside constructs. Check the documented subset of defaults and grants rather than assuming full Storm compatibility.
