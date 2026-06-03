# V2 Gotchas, Compile-Error Rules, Asset Modes, Tooling Integration

This file consolidates everything the Beta Guide doesn't make obvious — the rules that bite during build/edit. When in doubt about a field name or shape, mirror `canonical-example.md`.

## Project file layout

```
my-agent-network-project/
├── agent-network.yaml      # registry + context + brokers
├── exchange.json           # Anypoint Exchange asset metadata + variables
└── brokers/
    └── <broker-id>.agent   # Agent Script — one graph per file
```

For multi-broker projects: one `.agent` per broker, each referenced from its own entry under `brokers:`.

## Top-level YAML markers

- `agentNetwork: 2.0.0` — unquoted (per Beta Guide). (V1 used `schemaVersion: 1.0.0`.)
- `exchange.json` `"classifier": "agentic-network"` — V2. (V1 used `agent-network`.)
- `info.version` accepts non-semver strings (`v1` works in canonical).
- `policies` are part of V2 schema even though the MCP server's configure-instructions template (PR 1564) does not surface them in initial build. Use Workflow B.4 to add later.

## Agent Script `.agent` file

### Dialect line

```text
# @dialect: AGENTFABRIC=0.1-BETA
```

`0.1-BETA` is what canonical uses today. The V2 sample template references `1.0-BETA` as forward-looking — match canonical until your runtime accepts the newer one.

### `config:` block — minimal canonical form

```text
config:
  agent_name: "it-help-investigation"     # REQUIRED, kebab-case
  default_llm: @llm.gemini_flash          # optional
```

`agent_name` is required and MUST be kebab-case. Do NOT add `name` or `id`. `label` and `description` are optional.

### LLM `kind:` casing

Exact case: `"Gemini"` or `"OpenAI"`. Azure OpenAI deployments use `kind: "OpenAI"` in the `.agent` LLM block; `AzureOpenai` only appears at `registry.llms.<id>.metadata.platform`.

---

## Compile-error rules — action invocation

These bite the most.

### A2A actions

**Definition:** only `target` and `kind`. NO `inputs:`. A2A tools accept only a `message` parameter (plain string).

```text
actions:
  help_center_agent:
    target: "a2a://helpCenterAgentConnection"
    kind: "a2a:send_message"
```

**Inside `subagent`/`orchestrator` `reasoning.actions`:** bare reference. Adding `with message = ...` is a compile error. The LLM picks the message at runtime.

```text
reasoning:
  actions:
    search_help: @actions.help_center_agent      # ✓ bare reference
    # search_help: @actions.help_center_agent
    #   with message = "..."                      # ✗ COMPILE ERROR
```

**Inside `executor` `do:` `run`:** REQUIRES `with message = <value>` where `<value>` is string literal, `@reference`, or concatenation.

```text
do: ->
  run @actions.help_center_agent
    with message = "Search for: " + @generator.classify.output.topic
```

### MCP actions

**Definition:** `target`, `kind: "mcp:tool"`, `tool_name`, optional `inputs:`. Only declare `inputs:` if you actually know the parameter names and types.

```text
actions:
  escalate_ticket:
    target: "mcp://escalationMcpConnection"
    kind: "mcp:tool"
    tool_name: "escalate"
    inputs:
      ticket_id: string
      severity: string
```

**`with` parameters must be declared in `inputs:`** — passing an undeclared `with` parameter is a compile error. If `inputs:` is omitted, the invocation MUST have zero `with` parameters (other than `http_headers`).

**Slot-fill (`...`)** is only valid inside `reasoning.actions` MCP `with` clauses. Never in executor `run`.

```text
reasoning:
  actions:
    update_ticket: @actions.update_jira_ticket
      with ticket_id = @generator.classify.output.ticket_id     # ✓ declared input
      with http_headers = {"Authorization": @request.headers["Authorization"]}     # ✓ implicit
    send_slack: @actions.send_slack_message
      with message = ...                                        # ✓ slot-fill (MCP only)
```

### `http_headers` — the ONE implicit parameter

Every action automatically accepts an optional `http_headers` parameter (an object). Never declare in `inputs:`. Use to propagate auth or correlation IDs.

## Other compile-error rules

- **Trigger `on_message:`** must be a fixed `transition to`. Conditional routing inside `on_message:` is a compile error — use a `router` node.
- **Router `on_exit:` with `transition to`** is a compile error. Routers transition exclusively via `routes` and `otherwise`.
- **`a2a.task` / `a2a.message` / `a2a.textPart` / `uuid()`** are FUNCTIONS, not references — do NOT prefix with `@`.

## Echo `state` enum

The `state` field in `a2a.task({state: ...})` MUST be exactly one of:
`submitted`, `working`, `input-required`, `completed`, `failed`, `canceled`, `rejected`. Custom values are rejected.

## Expression syntax — three forms

| Form | Where | Example |
|---|---|---|
| `{!@<reference>}` | Inside plain string literals (`prompt:`, `reasoning.instructions:`) | `"Severity is {!@generator.classify.output.severity}"` |
| Direct `@reference` | Inside `+` concatenation, in `with` values, inside `a2a.*()` echo helpers | `"Ticket " + @generator.classify.output.ticket_id` |
| Slot-fill `...` | `reasoning.actions` MCP `with` clauses ONLY | `with message = ...` |

**Inside echo `a2a.task() / a2a.message() / a2a.textPart()`, references are direct.** `{!@...}` does NOT apply inside echo helpers.

## Subagent vs orchestrator — the decision

```
Does this node use any actions OR require human-in-the-loop?
  ├─ NO  → generator (single LLM call, no actions, no HITL)
  └─ YES
       └─ Does this node coordinate MULTIPLE actions toward a compound goal?
            ├─ YES → orchestrator
            └─ NO  → subagent (default)
```

Single-action node calling one A2A agent → **`subagent`**. NOT `orchestrator`. The Beta Guide's older guidance ("any A2A → orchestrator") is wrong for V2 — node type is determined by *coordination scope*, not protocol.

`subagent` and `orchestrator` MUST have ≥1 action OR HITL. Otherwise use `generator`.

**HITL is built into `subagent`** via the runtime's `input-required` A2A response. Don't model clarification as an external `subagent → router on needs_clarification → executor → echo` flow.

## Operational parameters — set them, don't leave defaults

Two optional fields on every `subagent`/`orchestrator`:

- **`reasoning.max_number_of_loops`** — caps the agent loop. Default 25. Lower for tight loops: classification subagents fine at 3, multi-action orchestrators usually 5–10. Default 25 risks runaway token spend on bad inputs.
- **`reasoning.task_timeout_secs`** — wall-clock cap. No documented default; set when calling slow downstream agents.

```text
orchestrator crossPlatformTriage:
  reasoning:
    instructions: -> | {!@request.payload.message.parts[0].text}
    actions: { ... }
    max_number_of_loops: 5
    task_timeout_secs: 60
```

## `outputs:` placement — different per node type

- **`generator`**: at node top level (sibling of `prompt:`).
- **`subagent` / `orchestrator`**: nested inside `reasoning:` (sibling of `reasoning.instructions` and `reasoning.actions`).

Wrong placement is a compile error.

## Authentication kind casing

Three V2 sources disagree slightly. **When in doubt, copy canonical.**

| Auth kind | Canonical example | V2 MCP server template (PR 1564) | V2 Beta Guide |
|---|---|---|---|
| API key | `apiKey` | `api-key` | `apiKey` |
| API key client credentials | (not used) | `api-key-client-credentials` | `apikey-client-credentials` |
| OAuth2 client credentials | `oauth2-client-credentials` | `oauth2-client-credentials` | `oauth2-client-credentials` |
| OAuth2 OBO | `oauth2-obo` | (not listed) | `oauth2-obo` |
| Basic auth | (not used) | `basic` | `basic` |
| In-task authorization code | (not used) | (not listed) | `in-task-authorization-code` |

Default: `apiKey`, `oauth2-client-credentials`, `oauth2-obo`. For kinds canonical doesn't use, prefer Beta Guide casing.

`in-task-authorization-code` is documented in Beta Guide for OAuth2 step-up but not in canonical or MCP template. Don't emit unless explicitly requested.

## Variables block — the rules

```text
variables:
  response_message: string
  response_state: string
```

- Only declare for **cross-path shared state** (e.g., `response_message` written by multiple paths and read by a shared echo).
- Do NOT mirror node outputs (`@<nodeType>.<nodeId>.output`) or globally-accessible request data — reference directly.
- Never declare a variable that isn't both set and read by ≥1 node.
- Syntax: `<name>: <type>` with optional indented `description:`. NO `mutable`, NO defaults like `= ""`.

## Supported LLMs — model IDs change fast

Always look up current production model IDs at provider docs:
- Gemini: <https://ai.google.dev/gemini-api/docs/models>
- OpenAI: <https://developers.openai.com/api/docs/models>

Canonical IT Help Network uses `gemini-2.5-flash` and `gpt-5.4`. Beta Guide lists older OpenAI IDs (`gpt-5.2`, `gpt-5.2-pro`, `gpt-5-mini`, `gpt-5-nano`, `gpt-5`). Confirm with user before emitting `gpt-5.4` — it's not in published Beta Guide list. When in doubt, emit `gpt-5.2-pro` (Pro tier) or `gpt-5-mini` (mini tier).

---

## RULE-ASSET-MODE — inline vs Exchange

Every asset is in **one of two modes**. Each asset's mode is independent — a single project can mix.

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

## Asset selection — identify, search, register, bind

### Stage 1: Identify the need

| Need | Asset type |
|---|---|
| Delegate to another autonomous agent that owns its logic | A2A agent |
| Call a specific API/function | MCP server tool |
| Classify, summarize, generate, judge | LLM |

Prefer MCP for single API calls (cheaper, faster). Prefer A2A for multi-step delegation (autonomous reasoning).

### Stage 2: Search Exchange

If `search_asset` is available, call it per asset need. Search **Private Exchange first**; only fall back to Public if Private has no match. Confirm before pulling Public.

| Asset type | `assetFilters` |
|---|---|
| LLM | `["llm"]` |
| MCP server | `["mcp"]` |
| A2A agent | `["agent"]` |

If `search_asset` isn't available, ask: *"Do you have this in Exchange already, or should I register a placeholder?"*

```
search_asset → found in Private?       → Case A (Exchange, Private)
            → found only in Public?    → Case B (confirm, then Exchange, Public)
            → not found anywhere?      → Case C (inline placeholder)
```

**Case C placeholder:** create a registry entry with `default: ""` for URLs/secrets in `exchange.json`. Flag clearly. Example:

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

### Stage 3: Register

Per asset, atomically:
1. Inline OR Exchange definition (per RULE-ASSET-MODE).
2. Add any new `${...}` variables to `exchange.json.metadata.variables`. Mark URLs `secret: false`. API keys / passwords / OAuth secrets `secret: true`.
3. Confirm with user before next asset.

**Naming conventions** (from canonical):
- Asset id: camelCase (`helpCenterAgent`, `escalationMcp`, `gemini`).
- Connection id: `<assetId>Connection`.
- Variables: `<assetId>.url`, `<assetId>.apiKey`, etc.

### Stage 4: Bind to nodes

**Cap at 4 actions per LLM-powered node** (guideline). Beyond 4, hallucination rates climb. Move actions to a downstream `executor`, split via `router`, or drop unused actions. Don't refuse a 5th if user accepts the risk.

**CR-18 nuance — least privilege.** Hard business constraints are enforced by **router conditions**, not prompts.

- **Irreversible / high-stakes mutations** (escalate, delete, restart, send-to-customer) MUST be in `executor` nodes gated by `router`. NEVER on `subagent`/`orchestrator` — the LLM could invoke them bypassing the router.
- **Idempotent updates** integral to the compound goal (status updates, ticket updates, log entries) MAY be on a `subagent`/`orchestrator`. Canonical IT Help does this: `crossPlatformTriage` (orchestrator) has `update_jira_ticket` because updating Jira is part of the compound triage goal. **It does NOT have `escalate_ticket`** — that's irreversible and lives on executors gated by routers.

The line: irreversible/high-stakes (executor-only, router-gated) vs idempotent/domain-natural (orchestrator OK).

**LLM tier per node:**

| Tier | When |
|---|---|
| Pro | Multi-step orchestration, ambiguous classification, ≥3 actions |
| Flash / mini | Single-step classification, summary, formatting, templated text |

Use `default_llm` for common case. Override per-node only on exceptions.

**Action aliases** inside `reasoning.actions` — short readable names (`search_help`, `slack_update`, `escalate`). The alias is what the LLM sees — clear aliases reduce hallucinated tool calls.

### Worked example — IT Help Network

| Node | Type | Actions | Count | LLM |
|---|---|---|---|---|
| `classifySeverity` | generator | (none) | 0 | gemini_flash (default) |
| `crossPlatformTriage` | orchestrator | help_center_agent, license_procurement_agent, update_jira_ticket | 3 | openai_gpt (override) |
| `helpSummary` / `licenseSummary` | generator | (none) | 0 | gemini_flash |
| `escalateTicket` / `escalateUnresolved` | executor | escalate_ticket | 1 | – |

Why `crossPlatformTriage` is orchestrator: coordinates **multiple** actions (search → check license → update ticket) toward a compound goal.

Why `classifySeverity` is generator: pure classification, no actions, no HITL.

Why `escalate_ticket` is on executors and NOT orchestrator: irreversible mutation, gated by router (severity for high-priority, resolution for unresolved).

### Common mistakes

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
- Putting hard constraints in prompts (use router conditions on identifying attributes — service name + recommended action — not derived judgment flags like `requires_approval`).
- Modeling HITL as external `subagent → router → executor` (subagents have built-in HITL).
- Using `{!@...}` template syntax inside `a2a.*()` echo helpers.
- Auto-inserting missing assets during validation (ask the user to fix or remove).

---

## Tooling integration — CLI-first, MCP-fallback

The skill prefers the **Anypoint CLI Agent Fabric plugin** for validate/publish/deploy because it's portable (every host, including CI/CD), official MuleSoft, and stays in sync with platform changes. The MuleSoft MCP server (auto-installed in ACB/Vibes) is the fallback — same operations, host-mediated auth.

### Capability matrix

| Capability | Step | CLI command (preferred) | MCP tool (fallback) | If neither |
|---|---|---|---|---|
| Scaffold project | 0 | *(skip — skill writes scaffold)* | *(skip — `create_agent_network_project`)* | — |
| Configure YAML | 0–5 | *(skill IS the experience)* | *(skip — `configure_agent_network_yaml` returns a duplicate prompt template)* | — |
| Search Exchange | 1, 2 | `anypoint-cli-v4 exchange asset search` | `search_asset` | Ask user for `groupId`/`assetId`/`version` |
| Validate / build | 6 | `agent-network project build` | `validate_project` | Structural checklist + doc link |
| Publish to Exchange | 7 | `agent-network project publish` | `publish_agent_network_assets` | Doc link |
| Deploy to runtime | 8 | `agent-network project deploy` | `deploy_agent_network` | Doc link |
| Set up gateways | one-time | `agent-network setup gateways` | *(no MCP equivalent)* | Doc link |

### Detection

In order at each integration point:

1. **CLI:** `command -v` the relevant binary —
   - `anypoint-cli-agent-fabric-plugin` for build/publish/deploy/setup-gateways
   - `anypoint-cli-v4` for Exchange search and any general Anypoint operation
2. **MCP:** Tool appears with prefix `mcp__mulesoft__` (e.g., `mcp__mulesoft__search_asset`).
3. **Neither:** Fall back to doc link or user prompt.

Don't probe filesystems for credentials — let CLI/host abstract auth.

### CLI auth (env vars only — never inline)

Both CLIs share Anypoint Platform auth via the same env vars:

- `ANYPOINT_CLIENT_ID` / `ANYPOINT_CLIENT_SECRET` (connected app credentials)
- `ANYPOINT_ORG` (org ID)
- `ANYPOINT_ENV` (environment name; defaults to Sandbox)
- `ANYPOINT_HOST` (defaults to `anypoint.mulesoft.com`; override for EU/Gov)

The general CLI also supports username/password auth via `anypoint-cli-v4 conf username <u>` / `conf password <p>` / `conf organization <o>`, but env-var/connected-app credentials are preferred for CI and shared sessions.

If env vars are unset and a CLI is being asked to do an auth-bearing action, point user at <https://docs.mulesoft.com/anypoint-cli/latest/auth>. Do not paste credentials, even temporarily.

Install:
- Agent Fabric plugin: `npm i mulesoft-anypoint-cli-agent-fabric-plugin` (renamed from `anypoint-cli-agent-fabric-plugin` — use `--force` if hitting an EXIST conflict).
- General CLI v4: `npm install -g anypoint-cli-v4`.

### Exchange search — Phase 1 preview, Phase 2 registration

**CLI (preferred):** `anypoint-cli-v4 exchange asset search --search "<query>" [--type <type>] [--organization <orgId>]`

- General CLI v4 type filters: `connector` | `rest-api` | `soap-api` | `template` | `example` | `custom` | `raml-fragment`. Agent-Fabric–specific types (LLM/MCP/Agent) aren't first-class here yet, so search broadly and filter the result list by name/description.
- Useful follow-ups: `anypoint-cli-v4 exchange asset describe <groupId>/<assetId>/<version>` to confirm details before registering.
- **Always pass `--organization <orgId>`** to scope to Private Exchange first. If no Private match, drop the flag (or pass the public org) — confirm with user before pulling Public.

**MCP fallback:** `search_asset` with `assetFilters: ["llm"]`, `["mcp"]`, `["agent"]`.

- At least one of `searchQuery` or `maxResults`.
- Optional: `statuses`, `organizationId`, `sharedWithMe`, `sortCriteria`, `ascending`, `exchangeScope` (Private / Public).
- Always include `assetFilters` — searching without narrows nothing.

If search returns no results, **don't invent the asset**. Tell the user: *"I didn't find a matching asset in Exchange. Want to provide a custom configuration, or skip?"*

### Step 6 — Validate / build

**CLI:** `anypoint-cli-agent-fabric-plugin agent-network project build --path <projectPath>`. Validates configuration via Maven wrapper, generates deployable artifact. Add `--debug` for verbose output. Exits non-zero on validation failure.

**MCP:** `validate_project` with `projectPath`. Returns schema/reference/expression findings.

Two kinds of fixes:
1. **Auto-fix safely** — formatting/indentation, missing required keys (e.g., `info.version`).
2. **Ask the user** — schema-required references missing. **Never auto-insert missing assets.**

Loop until clean.

### Step 7 — Publish

**CLI:** `anypoint-cli-agent-fabric-plugin agent-network project publish [--path <projectPath>] [--environment <env>] [--json]`.
- `assetVersion` is read from `exchange.json` (not a flag).
- `--json` returns asset URLs in parseable form for CI.

**MCP:** `publish_agent_network_assets` with `assetVersion` (semver), optional `projectPath`, `groupId`.

Prereqs (both paths): authenticated; valid `exchange.json` (no empty `default: ""` for `secret: true` variables); `assetVersion` set.

After publish, surface published asset URLs.

### Step 8 — Deploy

**One-time setup per private space:** `anypoint-cli-agent-fabric-plugin agent-network setup gateways --target-space <space>`. Defaults: ingress `agent-network-ingress-gw`, egress `agent-network-egress-gw`, space `agent-network-space`. Skip if gateways already exist.

**CLI deploy:** `anypoint-cli-agent-fabric-plugin agent-network project deploy [flags]`. Useful flags:
- `--environment <name>` (or `ANYPOINT_ENV`)
- `--target-space <name>` (defaults to `agent-network-space`)
- `--ingress-gw <name>` / `--egress-gw <name>` (defaults shown above)
- `--property k:v` (repeatable — env-specific secrets/config; e.g., `--property openai.apiKey:STAGING_API_KEY`)
- `--dont-wait-for-agent-network` (CI: return immediately)
- `--disable-tracing` (CI/test: skip telemetry)
- `--json` (parseable output)

**MCP:** `deploy_agent_network` with `projectPath`, `environmentName`, `privateSpaceName`, `ingressGatewayName`, `egressGatewayName`.

Prereqs (both paths): gateways exist in target space; successful publish (or assets in Exchange); secrets injected via `--property` or pre-filled `exchange.json`.

After deploy, surface URL/ID.

### Graceful degradation (neither CLI nor MCP)

The skill works in Cursor, standalone Claude Code, Codex — anywhere without MuleSoft tooling. When neither CLI nor MCP is available:

- **Phase 1/2 — search:** Ask the user directly. For Exchange-mode assets, get `groupId`/`assetId`/`version` and (for MCPs) transport kind + tool names. For unknown assets, register an inline placeholder per Case C. Tell the user: *"To search Exchange directly, install `anypoint-cli-v4` (`npm i -g anypoint-cli-v4`)."*
- **Step 6 — validation:** Run the structural checklist. Tell the user: *"To run MuleSoft's full schema validator, install the CLI plugin (`npm i mulesoft-anypoint-cli-agent-fabric-plugin`) or use the MuleSoft MCP server in your IDE. CI/CD reference: <https://docs.mulesoft.com/anypoint-code-builder/af-build-agent-networks-in-a-ci-cd-environment>."*
- **Step 7 — publish:** Point at <https://docs.mulesoft.com/anypoint-code-builder/af-publish-agent-network-assets>.
- **Step 8 — deploy:** Point at <https://docs.mulesoft.com/anypoint-code-builder/af-deploy-agent-network-targets>.

### What this skill must NOT do

- **Don't call `configure_agent_network_yaml` or CLI `create`** — both duplicate skill logic and overwrite the canonical scaffold.
- **Don't silently shell out** for auth-bearing actions without confirming env vars are set; never paste credentials inline.
- **Don't auto-deploy.** Step 6 ends at "validated"; 7 is publish; 8 is deploy. Both opt-in.
- **Don't auto-insert missing assets** when validation reports a dangling reference. Ask user.
- **Don't fabricate** MCP `tool_name`, asset IDs, or model IDs even when neither CLI nor MCP is present.
