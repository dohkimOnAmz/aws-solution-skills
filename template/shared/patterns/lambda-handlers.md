# Lambda Handler Patterns — <Solution Name>

> Replace this placeholder with full Lambda handler templates and the pitfalls each one avoids.

## Common helpers

```typescript
// backend/lambdas/_shared/http.ts
import type { APIGatewayProxyResult } from 'aws-lambda';

export function ok<T>(body: T): APIGatewayProxyResult {
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Headers': 'Content-Type,Authorization',
      'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
    },
    body: JSON.stringify(body),
  };
}

export function error(code: number, message: string): APIGatewayProxyResult {
  return { statusCode: code, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ error: message }) };
}

export function parseBody<T>(body: string | null): T {
  return JSON.parse(body ?? '{}') as T;
}
```

## <Handler 1>

```typescript
// Replace with actual handler pattern
```

## Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
