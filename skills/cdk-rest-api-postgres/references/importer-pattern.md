# Importer Pattern

Use this reference when stack code needs to attach to AWS resources that already exist outside this CDK stack.

Read `lib/constructs/api-importer.ts` before applying changes.

## Goal

Keep imported infrastructure references centralized through the repository's existing importer helper.

## Repository Shape

The stack already imports shared resources instead of scattering direct `from*` calls through `lib/core-stack.ts`.

Typical imports include:

- Cognito user pool
- S3 bucket
- API Gateway custom domain

## Baseline Example

```ts
import { Importer } from "./constructs/api-importer";

const userPool = Importer.getCognitoUserPoolById(this, process.env.CRM_USER_POOL_ID!);
const s3Bucket = Importer.getS3Bucket(this, process.env.S3_BUCKET!);

const domain = Importer.getApiGatewayDomainName(
    this,
    process.env.API_DOMAIN_NAME!,
    process.env.API_DOMAIN_NAME_ALIAS!,
    process.env.ZONE_ID!,
);
```

## Guidance

- Prefer `Importer` over inline `fromBucketName`, `fromUserPoolId`, or similar calls.
- Keep import logic close to stack composition, not in handlers or controllers.
- Only import what the stack actually needs. Do not add unused infrastructure references.
