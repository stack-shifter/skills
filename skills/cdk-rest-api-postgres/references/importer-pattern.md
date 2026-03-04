# Importer Pattern

Use this reference when stack code needs a reusable pattern for attaching to AWS resources that already exist outside the current CDK stack.

## Goal

Keep imported infrastructure references centralized through one helper or module.

## Common Shape

Stacks often import shared resources instead of scattering direct `from*` calls through stack code.

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

- If the repository already has an importer helper such as `Importer`, use or extend it.
- If it does not, generate one helper module instead of repeating inline `fromBucketName`, `fromUserPoolId`, or similar calls.
- Keep import logic close to stack composition, not in handlers or controllers.
- Only import what the stack actually needs. Do not add unused infrastructure references.
