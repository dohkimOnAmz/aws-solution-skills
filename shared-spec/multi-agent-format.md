# Multi-Agent Format Spec

Defines the entry-file format each tool (Amazon Quick · Kiro · Claude Code) requires so a single body of solution knowledge can drive all three.

## Core principle

**Write once, three tools consume it** — real knowledge lives in [`<skill>/shared/`](./shared-knowledge-pattern.md). Tool entry files are thin wrappers.

| Layer | Author once | Consumed by |
|---|---|---|
| `shared/reference/*` | Yes | All three tools |
| `shared/patterns/*` | Yes | All three tools |
| `shared/examples/*` | Yes | All three tools |
| `quick/SKILL.md` | Yes (thin) | Amazon Quick only |
| `kiro/steering.md` + `kiro/specs/*` | Yes (thin) | Kiro only |
| `claude-code/CLAUDE.md` + `claude-code/commands/*` | Yes (thin) | Claude Code only |

Tool entry files contain only **trigger / workflow / shared-reference list**. They never duplicate code blocks, architecture diagrams, or decision matrices.

---

## Format 1 — Amazon Quick (`quick/SKILL.md`)

### Frontmatter (required)

```yaml
---
name: <slug>                         # kebab-case, single-word identifier
display_name: <Title Case>           # surface label
description: >
  1-2 sentences shown in the catalog UI
trigger: "Single natural-language trigger sentence"
icon: 🧬                             # single emoji
inputs:                              # high-level inputs the user must answer
  - industry
  - channels
  - pii_fields
tools:                               # MCP / built-in tools used
  - aws_knowledge_mcp_server__aws_search_documentation
  - aws_knowledge_mcp_server__aws_read_documentation
  - aws_knowledge_mcp_server__aws_get_regional_availability
  - run_python
  - file_write
  - file_read
  - start_task
---
```

### Body structure

```markdown
# <Skill Name>

## Purpose
1-2 paragraphs — what is built, for whom

## Knowledge sources
- `shared/reference/architecture.md` — architecture decisions and rationale
- `shared/reference/decision-tree.md` — conditional logic
- `shared/reference/aws-services.md` — catalogs (services, models, pricing)
- `shared/reference/<solution-specific>.md`
- `shared/patterns/cdk-stacks.md`
- `shared/patterns/lambda-handlers.md`
- `shared/patterns/frontend-pages.md`
- `shared/examples/`

## Workflow

### Phase 1: Discovery (conversational requirements)
Sequential questions. Skip those already known.

⛔ **GATE 1**: summarize requirements, confirm with user.

### Phase 2: Architecture Design
- apply decision-tree.md
- compute cost from aws-services.md
- verify regional availability via AWS Knowledge MCP

⛔ **GATE 2**: present design as table/diagram.

### Phase 3: Code generation
Use shared/patterns to emit files incrementally.

### Phase 4: Validate
`cdk synth` must pass.

### Phase 5: Deploy
Deploy guide + output inventory.

## Generation rules
- Languages and runtimes (TypeScript / Node 20 / esbuild)
- Frontend stack (React + Tailwind + shadcn/ui)
- Naming conventions

## When to call MCP
| When | MCP | Call |
|---|---|---|
```

### Length target
**100-300 lines.** If longer, you have inlined patterns/code — move them into `shared/`.

---

## Format 2 — Kiro

Kiro pairs **Steering Rules + Spec**.

### 2-A. `kiro/steering.md` (always-on)

```markdown
# <Skill Name> — Kiro Steering Rules

## Identity
You are a specialized code-generation agent that builds <solution> from user requirements.
You produce production-ready CDK + Lambda + Frontend tailored to the user's industry.

## Knowledge sources
ALWAYS read these files before generating code:
- `shared/reference/architecture.md`
- `shared/reference/decision-tree.md`
- `shared/reference/aws-services.md`
- `shared/patterns/cdk-stacks.md`
- `shared/patterns/lambda-handlers.md`
- `shared/patterns/frontend-pages.md`
- `shared/examples/`

## MCP tools
- AWS Knowledge MCP — service availability, API constraints
- (optional) CloudFormation MCP — template validation

## Workflow

### When the user asks to create a new <solution> project:
1. Discover requirements
2. Design architecture
3. Generate code incrementally
4. Validate with `cdk synth`
5. Provide deploy instructions

### When the user asks to modify an existing project:
1. Read existing schema.yaml
2. Identify constructs to change
3. Apply edits while preserving infrastructure dependencies
4. Re-validate

## Hard Constraints
1. <constraint 1>
2. <constraint 2>
3. ...

## Code generation rules
- TypeScript + aws-cdk-lib v2
- Node 20+ Lambda, esbuild
- React + Tailwind + shadcn/ui (NOT Cloudscape)
- AWS SDK v3 modular imports
```

### 2-B. `kiro/specs/generate-<name>.md`

A spec mapping requirement collection → output inventory.

```markdown
# Spec: Generate <Solution> Project

## Inputs
| Field | Question |
|---|---|
| industry | What industry? |
| channels | What customer data channels? |
| ... | ... |

## Outputs
- `bin/app.ts`, `cdk.json`, `package.json`
- `lib/*-stack.ts` (per shared/patterns/cdk-stacks.md)
- `backend/lambdas/*/handler.ts` (per shared/patterns/lambda-handlers.md)
- `frontend/src/{pages,components,api}` (per shared/patterns/frontend-pages.md)
- `scripts/{deploy,destroy,check-prerequisites}.sh`
- `config/schema.yaml`

## Validation
- `npm run synth`
- AWS Knowledge MCP cross-check on referenced APIs / IAM
- evals/<domain>-scenario.md fixtures
```

### Length targets
- `steering.md`: 80-200 lines
- `specs/<name>.md`: 100-250 lines

---

## Format 3 — Claude Code

### 3-A. `claude-code/CLAUDE.md`

Project-root guidance. **Hard Constraints are mandatory.**

```markdown
# <Skill Name>

You are a specialized code-generation agent that builds <solution> systems.
You create complete, deployable CDK projects tailored to specific industry requirements.

## Knowledge files
Read these before generating code:
- `shared/reference/architecture.md`
- `shared/reference/decision-tree.md`
- `shared/reference/aws-services.md`
- `shared/reference/constraints.md`
- `shared/patterns/cdk-stacks.md`
- `shared/patterns/lambda-handlers.md`
- `shared/patterns/frontend-pages.md`
- `shared/examples/`

## MCP servers
Configure in `~/.claude/mcp_servers.json`. Use AWS Knowledge MCP to:
- Verify regional service availability
- Check API constraints
- Validate CDK construct properties

## Commands
- `/generate-<name>` — Interactive project generation (commands/generate-<name>.md)

## Code generation rules
- Infrastructure: TypeScript + aws-cdk-lib v2
- Lambda: Node 20+ TypeScript, esbuild
- Frontend: React 18 + Vite + Tailwind v3 + shadcn/ui (NO Cloudscape)
- Testing: jest + CDK assertions
- `cdk synth` must pass before marking a task complete

## Hard Constraints
1. <constraint — one line each, max ~10 items>
2. ...
```

### 3-B. `claude-code/commands/generate-<name>.md`

```markdown
# /generate-<name> — Generate <Solution> Project

Interactive command to create a complete <solution> system from scratch.

## Usage
/generate-<name>

## Workflow

### Step 1: Collect requirements
Ask the user (skip what is already known): <numbered question list>

### Step 2: Confirm design
Summary table → wait for user confirmation.

### Step 3: Generate project
Create files in this order: <ordered list — references shared/patterns/>

### Step 4: Validate
Run `cdk synth`.

### Step 5: Provide deploy instructions
Include post-deploy verification steps.
```

### Length targets
- `CLAUDE.md`: 80-200 lines
- `commands/<name>.md`: 50-150 lines

---

## Common across all three

| Item | Consistent rule |
|---|---|
| **Workflow phases** | Discovery → Design → Generate → Validate → Deploy |
| **Gate pattern** | User confirmation between phases (especially Design — region/cost) |
| **MCP usage** | AWS Knowledge MCP recommended — verify model IDs and regional availability at runtime |
| **Hard Constraints** | Each tool entry file lists them (e.g. CP `_profileId` reservation) |
| **Frontend stack** | React + Vite + Tailwind v3 + shadcn/ui (NOT Cloudscape) |
| **Lambda stack** | Node 20 + esbuild + AWS SDK v3 modular |
| **CDK stack** | TypeScript + aws-cdk-lib v2 |

---

## Verification

After adding or modifying a skill:

```bash
# 1) Quick frontmatter sanity
yq -y . <skill>/quick/SKILL.md | head

# 2) shared/* references must be consistent across the three entry files
grep -l "shared/" <skill>/quick/SKILL.md <skill>/kiro/steering.md <skill>/claude-code/CLAUDE.md
# All three should reference the same set

# 3) Run eval scenarios in each tool
```

When the three entry files **point to different reference sets**, you have drift. That is the most common bug — always check.
