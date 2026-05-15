# Shared Knowledge Pattern

This document specifies how each skill MUST organize its knowledge so that all three agents (Amazon Quick · Kiro · Claude Code) consume the same source of truth.

## Core Principle

**Write knowledge once. Reference it from three entry points.**

```
                                 ┌─────────────────────────┐
                                 │   <skill>/shared/       │
                                 │   - reference/          │
   ┌──────────────┐              │   - patterns/           │              ┌──────────────┐
   │  quick/      │── reference ─│   - examples/           │── reference ─│  kiro/       │
   │  SKILL.md    │              │                         │              │  steering.md │
   └──────────────┘              └──────────┬──────────────┘              └──────────────┘
                                            │
                                  ┌─────────┴────────┐
                                  │  claude-code/    │
                                  │  CLAUDE.md       │
                                  └──────────────────┘
```

If the same code block, decision matrix, or service quota appears in two of the three entry-point files, that is a violation — move it to `shared/` and replace the duplicates with a path reference.

## What Goes in `shared/`

### `shared/reference/`

Stable knowledge that does not depend on the user's request:

| File | Contents |
|---|---|
| `architecture.md` | Solution architecture diagrams + WHY each stack exists |
| `decision-tree.md` | Conditional logic mapping user answers → component choices |
| `aws-services.md` | Service quotas, pricing, model catalogs, region availability summary |
| `constraints.md` | Known limits, gotchas, reserved names, region-specific behavior |
| `<solution-specific>.md` | Solution-specific deep dives (e.g. `calculated-attributes.md` for UCP) |

### `shared/patterns/`

Concrete code blocks the agent will adapt and emit:

```
shared/patterns/
├── cdk-stacks.md            Full CDK Construct/Stack source
├── lambda-handlers.md       Full Lambda handler source + pitfalls
├── frontend-pages.md        React + Tailwind + shadcn/ui pages
├── etl-transforms.md        (optional) Glue ETL / PySpark
├── bedrock-prompts.md       (optional) prompt templates
└── <domain>.md              other solution-specific patterns
```

Each pattern file MUST contain:
- **Working code**, not summaries — agents copy verbatim then adapt
- **WHY comments** — what trap is being avoided, why this design
- **Cross-layer mapping** — one feature flows through CDK + Lambda + Frontend; show all three

### `shared/examples/`

At least 2–3 concrete domain instantiations, ideally as `config/schema.yaml` snippets so the agent can offer them as starting points.

```
shared/examples/
├── travel.md
├── hotel.md
└── retail.md
```

## What Goes in Tool-Specific Files

Each entry-point file is a **thin wrapper** that:
1. States the tool-specific frontmatter or header (Quick frontmatter, Kiro steering identity, Claude Code CLAUDE.md preamble)
2. Lists the workflow phases
3. **References `shared/*` paths** — does not repeat their content

Tool-specific files MAY contain:
- Tool-specific MCP wiring instructions
- Hard Constraints list (1-line entries that link back to `shared/` for detail)
- Naming conventions and code-style rules (short)
- Slash command syntax (Claude Code only)

Tool-specific files MUST NOT contain:
- Code blocks longer than ~10 lines (those belong in `shared/patterns/`)
- Architecture diagrams (those belong in `shared/reference/architecture.md`)
- Decision matrices (those belong in `shared/reference/decision-tree.md`)
- Service quotas, model IDs, or pricing tables (those belong in `shared/reference/aws-services.md`)

## Reference Format

Use forward-slash relative paths from the skill root:

```markdown
- `shared/reference/architecture.md` — architecture decisions
- `shared/patterns/lambda-handlers.md` — handler patterns
```

All three entry-point files SHOULD list the **same set** of references. Drift between lists is the most common bug.

## Why This Matters (Real-World Drift Examples)

When tool files contain duplicated knowledge:

| Scenario | What goes wrong |
|---|---|
| New AWS region launches ML matching | Need to update three files; forgetting one means one tool gives wrong region advice |
| New Bedrock model ID released | Three files have the old ID; one tool generates code with stale model |
| Service quota raised by AWS | One file says "max 4 instances," other says "max 10" |
| Critical pitfall discovered (e.g. CP `_profileId` reservation) | Hard Constraint added to one tool, forgotten in others |

When knowledge is centralized in `shared/`:

| Scenario | What happens |
|---|---|
| Update region table in `shared/reference/aws-services.md` | All three tools immediately consume the new info on next invocation |
| Add a Hard Constraint | Add the 1-line entry to all three tool files (still 3 places, but each is just one line — code/explanation lives once in `shared/`) |

## Audit Checklist

Before merging changes to a skill:

```bash
SKILL=<skill-dir>

# 1. shared/ contains the actual knowledge?
find $SKILL/shared -name "*.md" | xargs wc -l | tail -1
# Expect total > 1500 lines for a real solution

# 2. Tool entry files are thin?
wc -l $SKILL/quick/SKILL.md $SKILL/kiro/steering.md $SKILL/claude-code/CLAUDE.md
# Each should be < 300 lines

# 3. No code-block duplication across tool files?
grep -A 5 "^\`\`\`typescript" $SKILL/quick/SKILL.md $SKILL/kiro/steering.md $SKILL/claude-code/CLAUDE.md | head
# Should return very little

# 4. All three tool files reference the same shared/* set?
for f in $SKILL/quick/SKILL.md $SKILL/kiro/steering.md $SKILL/claude-code/CLAUDE.md; do
  echo "=== $f ===" && grep -oE "shared/[^ \`]+\.md" $f | sort -u
done
# Lists should match (entries may be a subset, not contradictory)
```

## Migration: From a Monolithic Skill

If you have an existing single-file skill and want to convert it:

1. **Extract**: identify all code blocks, decision tables, architecture descriptions in your single file
2. **Move**:
   - Code blocks → `shared/patterns/<layer>.md`
   - Decision tables → `shared/reference/decision-tree.md`
   - Service tables → `shared/reference/aws-services.md`
   - Architecture overviews → `shared/reference/architecture.md`
3. **Replace**: in the original file, replace the moved content with a single line: `See \`shared/reference/architecture.md\``
4. **Duplicate the wrapper**: copy the now-thin file to all three tool entry points (Quick / Kiro / Claude Code), each with its tool-specific frontmatter/preamble
5. **Verify**: run the audit checklist above

## When to Bend the Rule

Acceptable duplication:

- **Hard Constraints summary list** — each tool entry file has the same 1-line items, since each tool needs them at-a-glance during workflow execution. The full detail behind each constraint still lives in `shared/`.
- **Workflow phase headings** — each tool may state the same five phase names (Discovery → Design → Generate → Validate → Deploy) in its own format, since the phases are tool-execution structure, not knowledge.

Unacceptable duplication: anything that could be wrong (a model ID, a quota, a code snippet, a pitfall explanation).
