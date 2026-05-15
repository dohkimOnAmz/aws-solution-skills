# AWS Services — Constraints & Quotas

> Replace this placeholder with service-specific quotas, regional availability, pricing, and (where applicable) Bedrock model catalog.

## <Service A>

| Item | Value | Note |
|---|---|---|
| Quota | <value> | Default; can request increase |
| Regional availability | <regions> | Verify via AWS Knowledge MCP |
| Pricing | <$> | |

## <Service B>

| Item | Value | Note |
|---|---|---|

## Bedrock Claude model catalog (when applicable)

| Model | Bedrock model ID | Context | Best for |
|---|---|---|---|
| Claude Opus 4.7 | `us.anthropic.claude-opus-4-7` | 1M | Highest reasoning |
| Claude Sonnet 4 | `anthropic.claude-sonnet-4-20250514-v1:0` | 200K | Balanced default |
| Claude Haiku 4.5 | `anthropic.claude-haiku-4-5-20251001` | 200K | Cheapest, fastest |

> Verify latest model IDs via AWS Knowledge MCP (`aws___search_documentation`) before generating code — this catalog ages quickly.

## Cost summary

| Tier | Components | Monthly estimate |
|---|---|---|
| Minimal (PoC) | <list> | ~$X |
| Full (with optional graph/cross-domain) | <list> | ~$Y |
