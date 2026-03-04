# Services Pattern

Use this reference when the repository needs reusable service classes for logging, storage, email, or mapping.

## Goal

Put external integrations and repeated translation logic into small classes that controllers can reuse.

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

export const loggerService = new LoggerService();
```

## Baseline Storage Service

```ts
export class StorageService {
    constructor(private readonly client: S3Client) {}

    async generateUploadUrl(key: string): Promise<{ url: string; bucketKey: string }> {
        const command = new PutObjectCommand({
            Bucket: process.env.S3_BUCKET!,
            Key: key,
        });

        return {
            url: await getSignedUrl(this.client, command, { expiresIn: 600 }),
            bucketKey: key,
        };
    }
}
```

## Guidance

- Construct SDK clients once in a shared composition module such as `src/app.ts`, then inject them into service classes.
- Keep service APIs task-oriented, for example `generateUploadUrl`, `sendMessage`, or `entityToResponseDto`.
- Use mappers for repeatable transformation rules, not for one-off controller reshaping.
- Do not embed response formatting in services unless the repository already follows that pattern.
