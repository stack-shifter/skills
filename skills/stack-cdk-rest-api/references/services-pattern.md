# Services Pattern

Use this reference when the repository needs services for domain workflows, external integrations, or repeated mapping.

## Goal

Keep business workflows and external integrations outside controllers, with persistence delegated to repositories. Reuse the project's
service style rather than requiring classes or separate mappers for trivial operations.

## Recommended Services

- `logger.service.ts` for console or structured logging behind a tiny interface
- `storage.service.ts` for presigned S3 uploads and downloads
- `services/messaging/` for SES or other notification providers
- `services/mapping/` for DTO-to-entity, entity-to-response, and query parsing helpers

## Baseline Logger

```ts
export class LoggerService {
	info(message: string): void {
		console.log(`[API INFO] ${message}`);
	}

	warn(message: string): void {
		console.log(`[API WARN] ${message}`);
	}

	error(message: string, error?: Error): void {
		console.log(`[API ERROR] ${message}`);
		if (error) console.error(error);
	}
}
```

## Guidance

- Construct SDK clients once in a shared composition module such as `src/app.ts`, then inject them into service classes.
- Keep service APIs task-oriented, for example `generateUploadUrl`, `sendMessage`, or `entityToResponseDto`.
- Use mappers for repeatable transformation rules, not for one-off controller reshaping.
- Do not embed response formatting in services unless the repository already follows that pattern.
- Name public operations descriptively, such as `findAssociationOptions`, instead of ambiguous names like `options`. Follow existing CRUD
  vocabulary rather than enforcing these illustrative names.
- Use named result models for public object return signatures. Map persistence rows explicitly to the public contract; do not accidentally
  expose storage-only fields.
- Await into descriptive locals before response parsing or return construction. For example, fetch `items`, parse them into
  `validatedItems`, then return the named result. Keep awaits within existing error-handling boundaries.
