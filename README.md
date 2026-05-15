# aws-solution-skills

> **Multi-agent AI Skills for AWS solution patterns.** Ship the *ability* to generate a solution, not a static template. One source of knowledge, three agent entry points: Amazon Quick · Kiro · Claude Code.

## What this repo is

Each top-level folder is a single AWS solution use case (e.g. Unified Customer Profile, Data Lake, Bedrock RAG) packaged as a **skill** with end-to-end code-generation capability. The user describes their industry and requirements in conversation; the skill produces CDK + Lambda + Frontend tailored to that input.

## Skill catalog

| Skill | Solution | Trigger (example) | Status |
|---|---|---|---|
| [`unified-customer-profile-skill`](./unified-customer-profile-skill/) | AWS Connect Customer Profiles + Entity Resolution + Bedrock | "Build me a unified customer profile system" | ✅ Stable |
| _(planned)_ `data-lake-skill` | S3 Tables + Glue + Athena + Lake Formation | "Build me a data lake" | 🚧 Planned |
| _(planned)_ `bedrock-rag-skill` | Bedrock Knowledge Bases + OpenSearch Serverless | "Build me an internal RAG system" | 🚧 Planned |

## Supported agents (multi-agent)

Every skill ships **three entry-point formats over one shared knowledge base**.

| Tool | Entry file | Use context |
|---|---|---|
| **Amazon Quick** | `<skill>/quick/SKILL.md` | Conversational + background tasks + MCP integration |
| **Kiro** | `<skill>/kiro/steering.md` + `<skill>/kiro/specs/*.md` | In-IDE project generation and modification |
| **Claude Code** | `<skill>/claude-code/CLAUDE.md` + `<skill>/claude-code/commands/*.md` | CLI slash commands for full-project generation |

All three reference the same files under **`<skill>/shared/`** — knowledge is written once.

## Standard directory layout (every skill)

```
<solution-name>-skill/
├── README.md                       ← skill overview and usage
├── quick/
│   └── SKILL.md                    ← Amazon Quick entry
├── kiro/
│   ├── steering.md                 ← Kiro Steering Rules
│   └── specs/
│       └── generate-<name>.md      ← Kiro Spec
├── claude-code/
│   ├── CLAUDE.md                   ← Claude Code project guidance
│   └── commands/
│       └── generate-<name>.md      ← /generate-<name> slash command
├── shared/                         ← ⭐ the actual knowledge — referenced by all three tools
│   ├── reference/                  ← decisions, constraints, model/service catalogs
│   ├── patterns/                   ← CDK / Lambda / Frontend / ETL code patterns
│   └── examples/                   ← industry-specific golden examples
└── evals/                          ← scenario-based verification
```

Full spec: [`shared-spec/skill-structure.md`](./shared-spec/skill-structure.md).

## Using a skill

### Amazon Quick
```
Settings → Capabilities → Skills → Import → <skill>/quick/SKILL.md
```

### Kiro
```bash
# Option A: copy into workspace .kiro/
cp -r <skill>/kiro/.kiro/ <your-workspace>/.kiro/

# Option B: copy the whole skill so steering can resolve shared/* paths (recommended)
cp -r <skill>/ <your-workspace>/.kiro/skills/<skill-name>/
```

### Claude Code
```bash
# Per-project (recommended)
cp <skill>/claude-code/CLAUDE.md <your-project>/CLAUDE.md
cp -r <skill>/claude-code/commands/ <your-project>/.claude/commands/
cp -r <skill>/shared/ <your-project>/.claude/skills/<skill-name>/shared/

# Or globally (available across projects)
cp -r <skill>/ ~/.claude/skills/<skill-name>/
```

## Adding a new skill

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) and the [`template/`](./template/) directory. A new skill must satisfy:

1. The standard directory layout — [`shared-spec/skill-structure.md`](./shared-spec/skill-structure.md)
2. Three tool entry points (Quick / Kiro / Claude Code) — [`shared-spec/multi-agent-format.md`](./shared-spec/multi-agent-format.md)
3. **All real knowledge in `shared/`**, tool files are thin wrappers — [`shared-spec/shared-knowledge-pattern.md`](./shared-spec/shared-knowledge-pattern.md)
4. At least one evaluation scenario in `evals/`

## MCP requirements (skill-common)

| MCP | Purpose | Required? |
|---|---|---|
| AWS Knowledge MCP | Service docs, regional availability, model IDs | Recommended (real-time verification) |
| CloudFormation MCP | Stack validation and deployment | Optional |
| Bedrock MCP | AI feature smoke-testing | Optional |

## Design principles

1. **Shared knowledge** — real content lives only in `<skill>/shared/`. Tool entry files only define entry point and workflow phases.
2. **Gate pattern** — Discovery → Design → Generate → Validate → Deploy. User confirmation between phases (especially Design, where region/cost are locked in).
3. **MCP-first** — never hard-code volatile catalogs (model IDs, region availability). Verify via MCP at runtime.
4. **Golden examples** — patterns are extracted from working reference projects, not idealized prose.
5. **Multi-agent parity** — the three entry files MUST reference the same `shared/*` set. Drift is the #1 bug.

## License

TBD (Amazon internal — pending external publication review).
