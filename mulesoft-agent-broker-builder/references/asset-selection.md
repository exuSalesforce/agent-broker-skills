# Asset Selection & Assignment

How to register assets and bind them to nodes in V2. The Beta Guide covers the schemas; this file covers the inline-vs-Exchange split, the cap rules, and the binding gotchas.

## RULE-ASSET-MODE — the two modes

Every asset is in **one of two modes**:

### Mode 1 — Inline (asset defined locally)

Used when the asset doesn't exist in Anypoint Exchange.

**Two places:**
- **a)** `registry.{agents,mcps,llms}.<id>` — asset definition.
- **b)** `context.connections.<connId>` with `ref.name: <id>` matching the registry id.

No `exchange.json.dependencies` entry. No `ref.namespace`.

### Mode 2 — Exchange (asset is a published dependency)

Used when the asset exists in Anypoint Exchange — Private (your org) or Public.

**Two places:**
- **a)** `exchange.json.dependencies` — `{ groupId, assetId, version, classifier, packaging: "zip" }`.
- **b)** `context.connections.<connId>` with `ref.name: <assetId>` AND `ref.namespace: <groupId>`. **No registry entry.**

Each asset's mode is independent. A single project can mix.

## Stage 1: Identify the need

For each step needing external work:

| Need | Asset type |
|---|---|
| Delegate to another autonomous agent that owns its logic | A2A agent |
| Call a specific API/function | MCP server tool |
| Classify, summarize, generate, judge | LLM |

Prefer MCP for single API calls (cheaper, faster). Prefer A2A for multi-step delegation (autonomous reasoning).

## Stage 2: Search Exchange

If `search_asset` is available, call it per asset need. Search **Private Exchange first**; only fall back to Public if Private has no match. Confirm before pulling Public assets.

| Asset type | `assetFilters` |
|---|---|
| LLM | `["llm"]` |
| MCP server | `["mcp"]` |
| A2A agent | `["agent"]` |

If `search_asset` isn't available, ask the user directly: *"Do you have this in Exchange already, or should I register a placeholder?"*

### Decision tree per asset

```
search_asset → found in Private?       → Case A (Exchange, Private)
            → found only in Public?    → Case B (confirm, then Exchange, Public)
            → not found anywhere?      → Case C (inline placeholder)
```

**Case A/B** (Exchange): pull `groupId`, `assetId`, `version`. Add to `exchange.json.dependencies` + `context.connections.<id>` with `ref.name` and `ref.namespace`. No registry entry.

**Case C** (placeholder): create a registry entry with `default: ""` for URLs/secrets in `exchange.json`. Flag clearly. Example:

```yaml
registry:
  agents:
    customAgent:
      info:
        label: Custom Agent (PLACEHOLDER — fill in URL and skills)
      metadata:
        protocol: a2a
        platform: Other
        card:
          a2a:
            protocolVersion: 0.3.0
            name: customAgent
            description: TODO
            url: ${customAgent.url}
            version: 1.0.0
            capabilities: { pushNotifications: false }
            defaultInputModes: [application/json, text/plain]
            defaultOutputModes: [application/json, text/plain]
            skills: []
```

When you finish the build, list every placeholder explicitly. **Never fabricate** asset IDs, URLs, or MCP tool names.

## Stage 3: Register the asset

Per asset, atomically:
1. Inline OR Exchange definition (per RULE-ASSET-MODE above).
2. Add any new `${...}` variables to `exchange.json.metadata.variables`. Mark URLs `secret: false`. Mark API keys / passwords / OAuth secrets `secret: true`.
3. Confirm with user before moving to next asset.

**Naming conventions** (from canonical example):
- Asset id: camelCase (`helpCenterAgent`, `escalationMcp`, `gemini`).
- Connection id: `<assetId>Connection`.
- Variables: `<assetId>.url`, `<assetId>.apiKey`, etc.

## Stage 4: Bind to nodes

### Rule 1 — `subagent` is the default; `orchestrator` is for compound goals

```
Does this node use any actions OR require HITL?
  ├─ NO  → generator
  └─ YES → does it coordinate MULTIPLE actions toward a compound goal?
          ├─ YES → orchestrator
          └─ NO  → subagent (default)
```

The Beta Guide's older guidance ("any A2A → orchestrator") is wrong for V2. Node type is determined by **coordination scope**, not protocol. Single-action with one A2A agent → `subagent`. Single-action with HITL clarification → `subagent`. Multi-action coordination (call A → B → C, react to outcomes) → `orchestrator`.

`subagent` and `orchestrator` MUST have ≥1 action OR HITL.

### Rule 2 — Cap at 4 actions per LLM-powered node (guideline)

Beyond 4 actions, hallucination rates climb. When a node would exceed 4:
- Move some actions to a downstream `executor`.
- Split the node into specialized nodes connected via a `router`.
- Drop unused actions.

Don't refuse a 5th if user accepts the risk.

### Rule 3 — Principle of least privilege (CR-18 nuance)

Hard business constraints are enforced by **router conditions**, not prompts.

**Irreversible / high-stakes mutations** (escalate, delete, restart, send-to-customer) MUST be in `executor` nodes gated by `router`. NEVER on a `subagent`/`orchestrator` — the LLM could invoke them bypassing the router.

**Idempotent updates** that are integral to the compound goal (status updates, ticket updates, log entries) MAY be on a `subagent`/`orchestrator`. The canonical IT Help Network does this: `crossPlatformTriage` (orchestrator) has `update_jira_ticket` because updating Jira with notes is part of the compound triage goal. **It does NOT have `escalate_ticket`** — that's irreversible and lives on executors gated by routers.

The line: irreversible/high-stakes (executor-only, router-gated) vs idempotent/domain-natural (orchestrator OK).

### Rule 4 — Pick the right LLM per node

| Tier | When |
|---|---|
| Pro | Multi-step orchestration, ambiguous classification, ≥3 actions |
| Flash / mini | Single-step classification, summary, formatting, templated text |

Use `default_llm` for the common case. Override per-node only on exceptions.

**Always look up current model IDs at provider docs:** Gemini at `ai.google.dev/gemini-api/docs/models`, OpenAI at `developers.openai.com/api/docs/models`. Don't paste a stale ID.

### Rule 5 — Action aliases

Inside `reasoning.actions`, give each action a short readable alias (`search_help`, `slack_update`, `escalate`). The alias is what the LLM sees — clear aliases reduce hallucinated tool calls.

### Rule 6 — `with` clauses (compile-error rules)

In `reasoning.actions`:
- **A2A**: bare reference. NO `with message =` — compile error.
- **MCP**: optional `with` clauses for declared `inputs:`. Slot-fill `with <param> = ...` allowed.

In executor `run`:
- **A2A**: REQUIRES `with message = <value>`.
- **MCP**: `with` clauses must reference declared `inputs:`. No slot-fill.

`http_headers` is implicit on every action — never declare in `inputs:`. Use to propagate auth: `with http_headers = {"Authorization": @request.headers["Authorization"]}`.

## Worked example — IT Help Network

| Node | Type | Actions | Count | LLM |
|---|---|---|---|---|
| `classifySeverity` | generator | (none) | 0 | gemini_flash (default) |
| `crossPlatformTriage` | orchestrator | help_center_agent, license_procurement_agent, update_jira_ticket | 3 | openai_gpt (override) |
| `helpSummary` / `licenseSummary` | generator | (none) | 0 | gemini_flash |
| `escalateTicket` / `escalateUnresolved` | executor | escalate_ticket | 1 | – |

Why `crossPlatformTriage` is orchestrator: coordinates **multiple** actions (search → check license → update ticket) toward a compound goal.

Why `classifySeverity` is generator: pure classification, no actions, no HITL. (If HITL clarification were needed, it'd be `subagent`.)

Why `escalate_ticket` is on executors and NOT orchestrator: irreversible mutation gated by router (severity router for high-priority, resolution router for unresolved).

## Common mistakes

- A2A for a single API call (use MCP).
- MCP for autonomous multi-step delegation (use A2A).
- Cramming everything into one orchestrator (split or move to executors).
- Declaring fabricated `inputs:` on MCP actions.
- Declaring `inputs:` on A2A actions (forbidden).
- Forgetting `${...}` variables in `exchange.json.metadata.variables`.
- Hardcoding URLs (always parameterize).
- Using `orchestrator` for a single-action node.
- Putting irreversible mutations on `subagent`/`orchestrator`.
- Mixing inline and Exchange registration for the same asset.
