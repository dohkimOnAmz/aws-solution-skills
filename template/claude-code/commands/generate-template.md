# /generate-<name> — Generate <Solution> Project

Interactive command to create a complete <solution> system from scratch.

## Usage
```
/generate-<name>
```

## Workflow

### Step 1: Collect requirements
Ask the user (skip what is already known):

1. **Industry**: airline, hotel, retail, finance, healthcare, telco, other
2. **Channels**: web, mobile, call center, OTA, POS, corporate, ...
3. **PII fields**: name, email, phone, DOB, loyalty number, address, ...
4. **Data types**: bookings, orders, visits, billing, ...
5. **KPIs**: annual spend, visit frequency, AOV, CLV, ...
6. **Matching strategy**: data quality (Rule-based vs ML)
7. **Optional features**: Knowledge Graph? Cross-Domain?
8. **Region**: deployment region and cost constraints
9. **ETL pipeline**: raw data normalization needed?
10. **LLM model**: which Bedrock Claude model? (verify latest IDs via AWS Knowledge MCP)

### Step 2: Confirm design
Present a summary table:

```
| Component | Choice | Rationale |
|---|---|---|
| Ingestion | CSV / Glue / Kinesis | based on volume |
| ER strategy | Rule / Advanced / ML | based on data quality |
| Graph | On / Off | based on need |
| Estimated cost | $X/month | breakdown |
```

Wait for user confirmation before proceeding.

### Step 3: Generate project
Create files in this order (reference `shared/patterns/`):

1. `package.json`, `tsconfig.json`, `cdk.json`, `jest.config.js`
2. `config/schema.yaml` (single source of truth)
3. `shared/` (project-internal types, validators, config-loader)
4. `lib/` CDK stacks
5. `bin/app.ts`
6. `backend/lambdas/` handlers
7. `backend/custom-resources/` Custom Resources for any AWS resources requiring SDK calls
8. `backend/glue-scripts/*.py` (if ETL enabled)
9. `frontend/` React app
10. `scripts/` deploy/destroy/check-prerequisites/update-frontend-env
11. `docs/` architecture / deployment / API reference

### Step 4: Validate
Run `cdk synth` and fix any errors. Iterate until clean.

### Step 5: Provide deploy instructions
- Prerequisites check
- Manual Console steps (if any)
- Deploy command
- Post-deploy verification, including any solution-specific lifecycle steps (data import, indexing wait, etc.)
