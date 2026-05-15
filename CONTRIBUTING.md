# Contributing — Adding a new Solution Skill

This document describes how to register a new solution use case under `aws-solution-skills/`.

## Scope guide (right size for one skill)

A single skill = **one identifiable AWS solution pattern**.

| ✅ Right size | ❌ Too narrow | ❌ Too broad |
|---|---|---|
| Unified Customer Profile (CP + ER + Bedrock) | "Create a Lambda function" | "Build AWS infrastructure" |
| Bedrock RAG (KB + OpenSearch + Agent) | "Create an S3 bucket" | "Build an AI system" |
| Data Lake (S3 Tables + Glue + Athena) | "Write an IAM policy" | "Build a data platform" |
| Real-time Analytics (Kinesis + Flink + OpenSearch) | "Provision a Kinesis stream" | "Event processing" |

Criteria:
- The user can invoke it with **one trigger sentence** ("Build me a RAG system")
- It produces a **CDK + Lambda + Frontend** full stack
- It accepts **industry/domain inputs** and customizes the output (validated by golden examples)

## Procedure

### 1. Create the directory

Copy the [`template/`](./template/) folder verbatim and rename:

```bash
cp -r template <solution-name>-skill
cd <solution-name>-skill
```

The skill folder name MUST be **kebab-case + `-skill` suffix** (e.g. `data-lake-skill`).

### 2. Verify the directory structure

Must match [`shared-spec/skill-structure.md`](./shared-spec/skill-structure.md). Missing folders cause one of the three tools to fail at entry.

```
<solution-name>-skill/
├── README.md
├── quick/SKILL.md
├── kiro/steering.md + specs/
├── claude-code/CLAUDE.md + commands/
├── shared/{reference,patterns,examples}/
└── evals/
```

### 3. Write `shared/` first (the most important step)

**All real knowledge lives in `shared/`.** Tool entry files (quick/kiro/claude-code) are thin wrappers that reference these paths.

Rules and rationale: [`shared-spec/shared-knowledge-pattern.md`](./shared-spec/shared-knowledge-pattern.md).

#### `shared/reference/`

| File | Contents |
|---|---|
| `architecture.md` | Solution architecture diagram + stack decisions and *why* |
| `decision-tree.md` | Mapping from user answers → component selection (e.g. data quality → ER strategy) |
| `aws-services.md` | Service quotas, pricing, model ID catalog |
| `constraints.md` | Known limits and gotchas (regional restrictions, reserved key names, etc.) |

#### `shared/patterns/`

```
shared/patterns/
├── cdk-stacks.md           ← Full CDK Construct/Stack source
├── lambda-handlers.md      ← Full Lambda handler source + pitfall guidance
├── frontend-pages.md       ← React + Tailwind + shadcn/ui pages
├── etl-transforms.md       ← (when applicable) PySpark / Glue ETL
└── <domain-specific>.md    ← er-strategies, bedrock-prompts, etc.
```

Each pattern file should contain:
- **Working code** (not summaries) — copy-paste-ready for an agent to adapt
- **WHY comments** — what design choice, what trap is being avoided
- **CDK + Lambda + Frontend mapping** — how a single feature spans all three layers

#### `shared/examples/`

- At least 2-3 industry/domain instantiations (e.g. travel, hotel, retail)
- Each as a `config/schema.yaml` snippet so the agent can offer it as a starting point
- Where possible, extract from actual working reference projects

### 4. Write tool entry files (thin)

[`shared-spec/multi-agent-format.md`](./shared-spec/multi-agent-format.md) has the exact frontmatter/header spec. Key sizes:

| File | Role | Recommended length |
|---|---|---|
| `quick/SKILL.md` | trigger + workflow + `shared/*` reference list | 100-300 lines |
| `kiro/steering.md` | identity + workflow + `shared/*` reference list | 80-200 lines |
| `kiro/specs/<name>.md` | Requirements → implementation spec | 100-250 lines |
| `claude-code/CLAUDE.md` | Project guidance + `shared/*` references + Hard Constraints | 80-200 lines |
| `claude-code/commands/<name>.md` | Slash command workflow | 50-150 lines |

**Rule**: never duplicate the same content across these files. Push it into `shared/` and reference.

### 5. At least one `evals/` scenario

```
evals/
└── <domain>-scenario.md   ← simulated user input + expected outputs
```

Format:
```markdown
## Scenario: <domain>

### User input
"We are a <industry> with <channels>. Data quality is <quality>..."

### Expected behavior
1. Discovery should collect <questions>
2. Design phase should propose <stacks>
3. Generation should produce <files>
4. <calculated attribute or KPI> should populate after deploy
```

### 6. Update README files

Both the skill's own README and the catalog table in the root [`README.md`](./README.md).

### 7. Verify

Import the skill into each tool and run the eval scenario:

```bash
# Claude Code
cp -r <skill>/ ~/.claude/skills/<skill>/
# In a fresh conversation, type the trigger phrase and confirm the workflow proceeds
```

## Modifying an existing skill

Edits inside `shared/` propagate to all three tools automatically (no need to touch tool files). This is the core benefit of the multi-agent layout.

## Common mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Code patterns written directly into Quick / Kiro / Claude Code files | Same knowledge in 3 places — drift inevitable | Move to `shared/patterns/`, replace with reference |
| Tool files only, no `shared/` | Knowledge fragmented, unmaintainable | Enforce template structure |
| Hardcoded volatile catalog (model IDs, prices) in code | Stale within months | Put in `shared/reference/` and add an MCP-verification note in workflow |
| Workflow does not invoke AWS Knowledge MCP | Agent generates code with stale IDs | Specify `aws___search_documentation` calls in Discovery or Design Phase |
