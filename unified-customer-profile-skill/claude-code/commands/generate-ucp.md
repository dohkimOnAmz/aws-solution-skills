# /generate-ucp — Generate Unified Customer Profile Project

Interactive command to create a complete UCP system from scratch.

## Usage
```
/generate-ucp
```

## Workflow

When the user invokes this command:

### Step 1: Collect Requirements

Ask the user (skip what's already known):

1. **Industry**: What industry? (airline, hotel, retail, finance, healthcare, telco, other)
2. **Channels**: What customer data channels exist? (web, mobile, call center, OTA, POS, corporate...)
3. **PII Fields**: What identifies a customer? (name, email, phone, DOB, loyalty number, address...)
4. **Data Types**: What transaction/event data to store? (bookings, orders, visits, billing...) → these become **CP child Object Types** (Reservation, Folio, Order, ...)
5. **KPIs**: Key customer metrics? → these become **Calculated Attributes** (e.g., TotalRevenueInAYear, MostRecentVisit). See `shared/reference/aws-services.md` and the calc-attr guide.
6. **Matching Strategy**: Data quality? (high uniformity → Rule-based / many variations → ML)
7. **Optional Features**: Knowledge Graph? Cross-Domain? (explain cost implications)
8. **Region**: Deployment region and cost constraints?
9. **ETL pipeline**: Will raw DB data need normalization + variant expansion before ER? (yes for demos with single-table guests; no if you already have unified channel records)
10. **LLM model**: Which Bedrock Claude model for AI features? See catalog in `shared/reference/aws-services.md`. Default: Opus 4.7 for ER rules + Haiku 4.5 for personalization. Verify latest IDs via AWS Knowledge MCP before generating.

### Step 2: Confirm Design

Present a summary table:
```
| Component | Choice | Rationale |
|-----------|--------|-----------|
| Ingestion | CSV / Kinesis | based on volume |
| ER Strategy | Rule / Advanced / ML | based on data quality |
| Graph | On / Off | based on need |
| Estimated cost | $X/month | breakdown |
```

Wait for user confirmation before proceeding.

### Step 3: Generate Project

Create files in this order:
1. `package.json`, `tsconfig.json`, `cdk.json`, `jest.config.js`
2. `config/schema.yaml` — from collected requirements. **MUST include**:
   - `object_types[]` with **NO `_profileId`** key. Use `GuestKey [PROFILE, UNIQUE]` on the parent and `GuestKey [PROFILE]` on children (`shared/patterns/lambda-handlers.md`).
   - `calculated_attributes[]` (mapped from KPI question — see `shared/reference/calculated-attributes.md`).
   - `features.ai.bedrock.model_id` and `personalization_model_id` from the LLM question (see `shared/reference/aws-services.md`).
   - `features.etl.mode` from the ETL question.
3. `shared/` — types, validators, config-loader
4. `lib/` — CDK stacks (reference `shared/patterns/cdk-stacks.md`).
   **Required Custom Resources**: `upsert-object-type` and `create-calculated-attributes` with a `SchemaRev` cache-buster in properties so handler-only changes still trigger CR re-runs.
5. `bin/app.ts`
6. `backend/lambdas/` — handlers (reference `shared/patterns/lambda-handlers.md`).
   **Required handlers**:
   - `profile-import/` — golden record → CP GuestProfile (Send to CP — Step 1)
   - `cp-data-import/` — PostgreSQL → CP child instances (Send to CP — Step 2). Self-invoke worker pattern; include cleanup helper.
   - `personalization/` — Bedrock invoke. Use `assembleProfile(profileId)` when ProfileId is given, `assembleFromGolden(goldenId)` when goldenId is given. **`assembleFromGolden` MUST call ListProfileObjects(Reservation/Folio) once it resolves the cpProfileId** — otherwise warnings will say "no concrete reservation data."
7. `backend/glue-scripts/build-er-input.py` — Raw → ER input ETL (reference `shared/patterns/etl-transforms.md`)
8. `backend/custom-resources/upsert-object-type/handler.ts` — exact mapping rules from `shared/patterns/lambda-handlers.md` (Target only on GuestProfile; Target omitted on children; AllowProfileCreation false on children).
9. `backend/custom-resources/create-calculated-attributes/handler.ts` — with Create+Update+ResourceNotFoundException fallback per `shared/reference/calculated-attributes.md`.
10. `frontend/` — React + Vite + Tailwind + shadcn/ui (reference `shared/patterns/frontend-pages.md`).
    **Required pages**: Dashboard, Ingestion (with "Run ETL" button), Matching Comparison, Accuracy, AI Rules (with model Select), **Send to CP** (Step 1+2 buttons), Profile View (with Calculated Attributes guide block), Profile Detail.
11. `scripts/` — `deploy.sh`, `destroy.sh`, `check-prerequisites.sh`, `update-frontend-env.sh`
12. `docs/` — architecture, deployment, API reference, **calc-attr-guide.md** (copy of `shared/reference/calculated-attributes.md`'s UI guide section)

### Step 4: Validate

Run `cdk synth` and fix any errors. Iterate until clean.

### Step 5: Provide Deploy Instructions

Output clear deployment steps including:
- Prerequisites check
- Manual Console steps (Connect CP activation)
- Deploy command
- Post-deploy verification — include the **Send-to-CP / Calculated Attribute** flow:
  1. ER 매칭 실행 (Matching Comparison page)
  2. **Send to CP — Step 1**: golden profiles import
  3. **Send to CP — Step 2**: Reservation/Folio import (3-10 min, background)
  4. Wait for Calculated Attribute Status → COMPLETED (a few minutes)
  5. Verify on Profile Detail page that calc attr values populate
  6. If empty: run the diagnostics in `shared/reference/calculated-attributes.md` "디버깅 체크리스트"
