---
name: <slug>
display_name: <Title>
description: >
  Replace with a 1-2 sentence description shown in the Quick catalog UI.
trigger: "<Single trigger sentence>"
icon: 🧬
inputs:
  - <input1>
  - <input2>
tools:
  - aws_knowledge_mcp_server__aws_search_documentation
  - aws_knowledge_mcp_server__aws_read_documentation
  - aws_knowledge_mcp_server__aws_get_regional_availability
  - run_python
  - file_write
  - file_read
  - start_task
---

# <Skill Name>

## Purpose
1-2 paragraphs on what this skill builds and for whom.

## Knowledge sources
- `shared/reference/architecture.md`
- `shared/reference/decision-tree.md`
- `shared/reference/aws-services.md`
- `shared/reference/constraints.md`
- `shared/patterns/cdk-stacks.md`
- `shared/patterns/lambda-handlers.md`
- `shared/patterns/frontend-pages.md`
- `shared/examples/`

## Workflow

### Phase 1: Discovery
List the questions the agent asks the user.

⛔ **GATE 1**: summarize requirements, await confirmation.

### Phase 2: Architecture Design
- Apply `decision-tree.md`
- Cost estimate from `aws-services.md`
- Verify regional availability via AWS Knowledge MCP

⛔ **GATE 2**: present design, await confirmation.

### Phase 3: Code generation
Reference `shared/patterns/*` to emit files in this order:
1. `package.json`, `tsconfig.json`, `cdk.json`
2. `config/schema.yaml`
3. `lib/*-stack.ts`
4. `backend/lambdas/*/handler.ts`
5. `frontend/src/...`
6. `scripts/*.sh`

### Phase 4: Validate
`cdk synth` must pass.

### Phase 5: Deploy
Provide step-by-step deployment guide.

## Generation rules
- TypeScript + aws-cdk-lib v2
- Node 20+ Lambda, esbuild
- React + Tailwind + shadcn/ui (NOT Cloudscape)
- AWS SDK v3 modular imports
- Korean / English follows the user's language

## When to call MCP
| When | MCP | Call |
|---|---|---|
| Region availability check | AWS Knowledge | `aws_get_regional_availability` |
| Latest model ID lookup | AWS Knowledge | `aws_search_documentation` |
| CDK construct verification | AWS Knowledge | `aws_read_documentation` |
