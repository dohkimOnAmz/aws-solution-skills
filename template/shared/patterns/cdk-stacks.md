# CDK Stack Patterns — <Solution Name>

> Replace this placeholder with full, working CDK code blocks the agent will adapt.

## Foundation Stack

```typescript
// lib/foundation-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as kms from 'aws-cdk-lib/aws-kms';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class FoundationStack extends Construct {
  public readonly kmsKey: kms.Key;
  public readonly dlq: sqs.Queue;

  constructor(scope: Construct, id: string, props: { projectName: string }) {
    super(scope, id);
    this.kmsKey = new kms.Key(this, 'Key', {
      alias: `alias/${props.projectName}`,
      enableKeyRotation: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });
    this.dlq = new sqs.Queue(this, 'Dlq', {
      queueName: `${props.projectName}-dlq`,
      encryption: sqs.QueueEncryption.KMS,
      encryptionMasterKey: this.kmsKey,
    });
  }
}
```

## <Solution-specific Stack>

```typescript
// Replace with the actual stack patterns for this solution
```

## Cross-stack wiring conventions

- All resources prefixed with `${projectName}-`
- KMS key passed from FoundationStack to all data resources
- DLQ shared across async Lambdas
- Outputs: `ApiUrl`, `Region`, plus solution-specific endpoints
