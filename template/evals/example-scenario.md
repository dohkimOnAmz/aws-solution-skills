# Scenario: <industry>

## User input
"We are a <industry>. We have <channels>. Data quality is <quality>. ..."

## Expected behavior

### Phase 1 — Discovery
The agent should ask:
- <question 1>
- <question 2>

### Phase 2 — Design
The agent should propose:
- <stack 1>
- <stack 2>
- Estimated cost: $X/month

### Phase 3 — Generate
Output should include:
- `bin/app.ts`
- `lib/<stacks>.ts`
- `backend/lambdas/<handlers>/handler.ts`
- `frontend/src/pages/<pages>.tsx`
- `config/schema.yaml`

### Phase 4 — Validate
- `cdk synth` passes
- AWS Knowledge MCP verifies regional availability

### Phase 5 — Deploy verification
- <solution-specific post-deploy checks>
