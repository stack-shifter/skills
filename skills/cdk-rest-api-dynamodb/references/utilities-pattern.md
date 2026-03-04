# Utilities Pattern

Use this reference when the repository needs shared helpers that do not belong in handlers, repositories, or service integrations.

## Goal

Centralize reusable response, error, and helper logic so DynamoDB controllers and handlers stay consistent.

## Recommended Utilities

- `rest-result.ts` for API Gateway responses
- `errors.ts` for application-specific error classes
- `status-code.ts` for named HTTP status constants
- cursor helpers for encoding and decoding pagination state
- key helpers when multiple repositories share the same PK/SK prefix conventions

## Baseline `RestResult` Rules

- set consistent JSON and CORS headers
- expose success and error constructors instead of repeating inline response objects
- prefer repository-agnostic error helpers for not found, validation, unauthorized, forbidden, and internal server errors

## Guidance

- Prefer one shared response helper over duplicated `APIGatewayProxyResult` literals.
- Put pagination cursor encoding in a utility when several repositories share it.
- Add key helpers only when a PK or SK convention is used in multiple repositories; otherwise keep key construction local to the repository.
- Keep utility functions pure where possible.
