# Importer Pattern

The Storm examples apply when the library is installed. For existing project constructs without Storm, use `general-cdk.md`; keep the same application layers.

Use this reference when stack code needs a reusable pattern for attaching to AWS resources that already exist outside the current CDK stack.

## Goal

Keep imported infrastructure references centralized through one helper or module.

## Common Shape

Use Storm `Importer` for supported resource imports. Verify its installed signatures before using them.

Typical imports include:

- Cognito user pool
- S3 bucket
- API Gateway custom domain

## Storm Example

Use the package importer for new infrastructure. Environment values below must be validated by the project configuration layer.

```ts
import { Importer } from "@stack-shifter/storm-stack";

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

- Use Storm `Importer`; use direct CDK imports for resources it does not support.
- Extract an importer module only when repetition or project conventions justify it. Use Storm’s existing methods before adding a project-specific helper.
- Keep import logic close to stack composition, not in handlers or controllers.
- Only import what the stack actually needs. Do not add unused infrastructure references.
