# Utilities Pattern

Use this reference when the repository needs shared runtime helpers that do not belong in controllers, repositories, or integration services.

Read these files first when they exist:

- `src/utilities/rest-result.ts`
- `src/utilities/errors.ts`
- `src/utilities/status-code.ts`
- pagination or cursor helpers under `src/utilities/`

## Goal

Centralize pure reusable helpers so handlers and controllers stay consistent.

## Recommended Utilities

- `rest-result.ts` for HTTP success and error responses
- `errors.ts` for application-specific error classes
- `status-code.ts` for named HTTP status constants
- cursor, encoding, date, or parsing helpers that are reused across repositories and controllers

## Baseline `RestResult` Rules

- set `Content-Type: application/json`
- set consistent CORS headers
- expose `Ok`, `Created`, `NoContent`, `BadRequest`, `NotFound`, `Unauthorized`, `Forbidden`, and `InternalServerError`
- include `fromDatabaseError(error)` when SQL constraint mapping is needed

## Guidance

- Prefer one shared response helper over repeated inline `APIGatewayProxyResult` objects.
- Put database-error translation in `RestResult` or a dedicated utility so controllers stay small.
- Keep utility functions pure where possible.
- Use custom error classes only when callers handle them differently from generic errors.
- Add cursor helpers when pagination rules are reused across repositories; keep the encoded cursor shape stable.
