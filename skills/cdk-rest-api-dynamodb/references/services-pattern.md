# Services Pattern

Use this reference when the repository needs reusable service classes for logging, storage, notifications, token verification, or mapping.

Read these files first when they exist:

- `src/services/logger.service.ts`
- `src/services/storage/storage.service.ts`
- `src/services/messaging/`
- `src/services/mapping/`
- `src/services/token.service.ts`

## Goal

Put external integrations and repeated translation logic into small reusable classes rather than duplicating it in handlers or repositories.

## Recommended Services

- `logger.service.ts` for console or structured logging
- `storage.service.ts` for presigned S3 uploads and downloads
- `services/messaging/` for SES or other outbound notification providers
- `services/token.service.ts` for in-Lambda JWT verification when API Gateway auth is not enough
- `services/mapping/` or `utilities/mappers/` for DTO, entity, and query translation

## Baseline Logger

```ts
export class LoggerService {
    info(message: string): void {
        console.log(`[API INFO] ${message}`);
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

- Construct AWS SDK clients once in `src/app.ts`, then pass them into services.
- Keep service methods named after domain tasks, not raw AWS commands.
- Use mappers when query parsing, key translation, or response mapping appears in more than one controller.
- Keep DynamoDB key-building logic close to repositories unless it is a cross-entity convention that deserves a shared helper.
