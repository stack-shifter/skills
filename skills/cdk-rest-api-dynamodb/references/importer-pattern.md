# Importer Pattern

Use this reference when you need a reusable pattern for importing existing AWS resources into CDK code.

## Goal

Keep imported infrastructure references centralized instead of scattering direct `from*` calls throughout stack code.

## Default Shape

- one helper module for imported resources
- thin wrapper methods around CDK `from*` APIs
- stable naming for imported constructs

## Baseline Example

```ts
import { Construct } from 'constructs';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as cognito from 'aws-cdk-lib/aws-cognito';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

export class Importer {
    static getDynamoDbTable(scope: Construct, tableName: string): dynamodb.ITableV2 {
        return dynamodb.TableV2.fromTableName(scope, 'TableImported', tableName);
    }

    static getCognitoUserPoolById(scope: Construct, userPoolId: string): cognito.IUserPool {
        return cognito.UserPool.fromUserPoolId(scope, 'UserPoolImported', userPoolId);
    }

    static getRestApiById(scope: Construct, apiId: string): apigateway.IRestApi {
        return apigateway.RestApi.fromRestApiId(scope, 'RestApiImported', apiId);
    }
}
```

## Guidance

- If the repository already has an importer helper, use or extend it.
- If it does not, introduce one helper class or module instead of scattering inline `from*` imports through stack code.
- Keep import logic close to stack composition, not in runtime code.
