# Spec: Generate <Solution> Project

## Inputs
The agent collects the following from the user (skipping anything already known):

| Field | Question |
|---|---|
| industry | What industry? (airline, hotel, retail, finance, healthcare, telco, other) |
| channels | What customer data channels exist? |
| pii_fields | What identifies a customer? |
| data_types | What transactional/event data is stored? |
| kpis | Key customer metrics? |
| matching_strategy | Data quality assessment (Rule-based vs ML) |
| optional_features | Knowledge Graph? Cross-Domain? |
| region | Deployment region and cost constraints |
| etl_pipeline | Will raw data require normalization before ER? |
| llm_model | Which Bedrock Claude model for AI features? |

## Outputs

The skill produces:
- `bin/app.ts`, `cdk.json`, `package.json`, `tsconfig.json`
- `config/schema.yaml` (single source of truth derived from inputs)
- `lib/*-stack.ts` — per `shared/patterns/cdk-stacks.md`
- `backend/lambdas/*/handler.ts` — per `shared/patterns/lambda-handlers.md`
- `backend/custom-resources/` — Object Type / Calculated Attribute custom resources
- `backend/glue-scripts/*.py` — per `shared/patterns/etl-transforms.md` (when ETL enabled)
- `frontend/src/{pages,components,components/ui,api,lib}` — React + Tailwind + shadcn/ui per `shared/patterns/frontend-pages.md`
- `scripts/{deploy,destroy,check-prerequisites,update-frontend-env}.sh`
- `docs/architecture.md`, `docs/deployment.md`, `docs/api.md`

## Validation
- `npm run synth` (CDK synth must pass)
- AWS Knowledge MCP cross-check on referenced APIs / IAM actions / model IDs
- Domain-specific evaluation under `evals/<domain>-scenario.md`

## Iteration
After Validate, the agent waits for user feedback and applies incremental changes by re-reading `config/schema.yaml` and patching the affected stacks/handlers/pages.
