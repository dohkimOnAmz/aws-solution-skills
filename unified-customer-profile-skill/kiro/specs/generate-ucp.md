# Spec: Generate Unified Customer Profile System

## Requirements

### Functional
- FR1: System shall ingest customer PII data from multiple channels
- FR2: System shall perform entity resolution to identify same customers across channels
- FR3: System shall support 3 matching types: Simple Rule, Advanced Rule, ML
- FR4: System shall provide AI-assisted rule improvement via Bedrock (Human-in-the-Loop)
- FR5: System shall store unified customer profiles with calculated attributes
- FR6: System shall expose REST API for profile queries and matching operations
- FR7: System shall provide React-based admin console for demo/operations
- FR8: [Optional] System shall support Knowledge Graph for network analysis
- FR9: [Optional] System shall support Cross-Domain matching for multi-domain scenarios

### Non-Functional
- NFR1: Infrastructure as Code via AWS CDK (TypeScript)
- NFR2: All data encrypted at rest (KMS) and in transit (TLS)
- NFR3: Cognito-based authentication for API and frontend
- NFR4: Dead letter queue for all async failures
- NFR5: Deployable with single script (`scripts/deploy.sh`)
- NFR6: Destroyable with single script (`scripts/destroy.sh`)

## Design

### Architecture Layers
1. **Foundation** — KMS key, SQS DLQ
2. **Storage** — S3 data bucket for CSV/Glue input
3. **Profiles** — Connect instance + Customer Profiles domain + Object Types + Calculated Attributes
4. **Matching** — Entity Resolution (Glue DB/Table, ER Workflows, DynamoDB results)
5. **Ingestion** — CSV upload (default) or Kinesis stream
6. **Auth** — Cognito User Pool + Hosted UI
7. **API** — API Gateway + Lambda handlers + Cognito authorizer
8. **[Optional] Graph** — Neptune cluster + VPC + GraphRAG Lambda
9. **[Optional] Cross-Domain** — Multiple CP domains + Platform-level aggregation

### Key Design Decisions
- Schema-driven: `config/schema.yaml` declares PII fields, object types, calculated attributes
- CDK reads schema at synth-time to generate resources dynamically
- ER input format: Glue table columns match schema PII fields
- Matching results stored in DynamoDB for dashboard display
- AI suggestions stored separately with approve/reject workflow

## Tasks

- [ ] Task 1: Create project scaffolding (bin/app.ts, package.json, configs)
- [ ] Task 2: Implement Foundation + Storage stacks
- [ ] Task 3: Implement Profiles stack (schema-driven object types + CAs)
- [ ] Task 4: Implement Matching stack (ER workflows, Glue, DynamoDB)
- [ ] Task 5: Implement Ingestion stack (CSV mode default)
- [ ] Task 6: Implement Auth stack (Cognito)
- [ ] Task 7: Implement API stack + Lambda handlers
- [ ] Task 8: Implement Frontend (React + Cloudscape)
- [ ] Task 9: Implement deploy/destroy scripts
- [ ] Task 10: Validate with `cdk synth`
- [ ] Task 11: [Optional] Graph stack
- [ ] Task 12: [Optional] Cross-Domain stack
