# <Solution Name> Skill — TEMPLATE

> Replace this README with skill-specific overview after copying. See [`../CONTRIBUTING.md`](../CONTRIBUTING.md).

## Trigger

```
"<Single natural-language phrase users say to invoke this skill>"
```

## Supported tools

| Tool | Entry file |
|---|---|
| Amazon Quick | `quick/SKILL.md` |
| Kiro | `kiro/steering.md` + `kiro/specs/generate-<name>.md` |
| Claude Code | `claude-code/CLAUDE.md` + `claude-code/commands/generate-<name>.md` |

## Outputs

This skill generates:
- `bin/app.ts`, `cdk.json`, `package.json`
- `lib/*-stack.ts` — CDK stacks
- `backend/lambdas/*/handler.ts` — Lambda handlers
- `frontend/src/` — React + Tailwind + shadcn/ui app
- `scripts/{deploy,destroy,check-prerequisites}.sh`
- `config/schema.yaml`

## MCP requirements

| MCP | Purpose | Required? |
|---|---|---|
| AWS Knowledge MCP | Service docs, regional availability | Recommended |
| CloudFormation MCP | Stack validation | Optional |

## Knowledge sources

All real knowledge lives in [`shared/`](./shared/):
- `shared/reference/` — architecture, decision tree, AWS services catalog, constraints
- `shared/patterns/` — CDK / Lambda / Frontend code patterns
- `shared/examples/` — industry-specific instantiations
