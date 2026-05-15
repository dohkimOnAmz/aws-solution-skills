# Architecture — <Solution Name>

> Replace this placeholder with the solution's architecture decisions and rationale.

## Overview

High-level diagram and description of the solution.

## Stack composition

| Stack | Purpose | Required? |
|---|---|---|
| Foundation (KMS, DLQ) | Encryption + dead-letter | Always |
| Storage (S3) | Data lake | Always |
| <stack-1> | <purpose> | Always / Optional |
| <stack-2> | <purpose> | Always / Optional |

## Why these choices

1. **<Decision 1>** — Why this approach over alternatives
2. **<Decision 2>** — Trade-offs and constraints
3. ...

## Data flow

```
Source → Ingestion → Processing → Storage → Consumption
```

Step-by-step description of how data moves through the solution.
