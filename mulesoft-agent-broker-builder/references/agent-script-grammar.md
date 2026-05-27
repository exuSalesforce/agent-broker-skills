# Agent Script & Agent Network V2 Grammar — Gotchas

The Beta Guide is the authoritative V2 syntax reference. Read it for full schemas, every field, every node type, every option. This file covers only the **gotchas, compile-error rules, and version-specific details** the Beta Guide doesn't make obvious — the things that bite skill users in practice.

When in doubt about a field name or shape, mirror the canonical example (`canonical-example.md`).

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

- `agentNetwork: "2.0.0"` — quoted in canonical example. (V1 used `schemaVersion: 1.0.0`.)
- `exchange.json` `"classifier": "agentic-network"` — V2. (V1 used `agent-network`.)
- `info.version` accepts non-semver strings (`v1` works in canonical).
- `policies` are still part of V2 schema even though the MCP server's configure-instructions template (PR 1564) does not surface them in initial build. Use `Workflow B Step 14` to add them later.

## Agent Script `.agent` file

### Dialect line

```text
# @dialect: AGENTFABRIC=0.1-BETA
```

`0.1-BETA` is what the canonical IT Help Network uses today. The V2 sample template references `1.0-BETA` as a forward-looking value — match the canonical until your runtime accepts the newer one.

### `config:` block — minimal canonical form

```text
config:
  agent_name: "it-help-investigation"     # REQUIRED, kebab-case
  default_llm: @llm.gemini_flash          # optional
```

`agent_name` is required and MUST be kebab-case. Do NOT add fields like `name` or `id`. `label` and `description` are optional.

### LLM `kind:` casing

Exact case: `"Gemini"` or `"OpenAI"`. Azure OpenAI deployments use `kind: "OpenAI"` in the `.agent` LLM block; `AzureOpenai` only appears at `registry.llms.<id>.metadata.platform`.

## Compile-error rules — action invocation

These are the rules most likely to bite. The Beta Guide states the action types but doesn't always make the invocation rules clear.

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

**Inside `executor` `do:` `run`:** REQUIRES `with message = <value>` where `<value>` is a string literal, `@reference`, or concatenation.

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

Every action automatically accepts an optional `http_headers` parameter (an object). Never needs to be declared in `inputs:`. Use it to propagate auth or correlation IDs.

## Other compile-error rules

- **Trigger `on_message:`** must be a fixed `transition to`. Conditional routing inside `on_message:` is a compile error — use a `router` node.
- **Router `on_exit:` with `transition to`** is a compile error. Routers transition exclusively via `routes` and `otherwise`.
- **`a2a.task` / `a2a.message` / `a2a.textPart` / `uuid()`** are FUNCTIONS, not references — do NOT prefix with `@`.

## Echo `state` enum

The `state` field in `a2a.task({state: ...})` MUST be exactly one of:
`submitted`, `working`, `input-required`, `completed`, `failed`, `canceled`, `rejected`.

Custom values are rejected.

## Expression syntax — three forms, used in different places

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

Two optional fields on every `subagent`/`orchestrator` are worth setting explicitly, especially during Phase 6 review:

- **`reasoning.max_number_of_loops`** — caps the agent loop. Default is 25. Lower for tight loops: classification subagents work fine at 3, multi-action orchestrators usually need 5–10. Leaving the default of 25 risks runaway token spend on bad inputs.
- **`reasoning.task_timeout_secs`** — wall-clock cap for the node. No documented default; set when calling slow downstream agents.

Example:
```text
orchestrator crossPlatformTriage:
  reasoning:
    instructions: -> | {!@request.payload.message.parts[0].text}
    actions: { ... }
    max_number_of_loops: 5
    task_timeout_secs: 60
```

Set these in Phase 4 (asset assignment) or Phase 6 (final review). The Beta Guide Appendix's Phase 6 calls these out as "key operational parameters."

## `outputs:` placement — different per node type

- **`generator`**: at node top level (sibling of `prompt:`).
- **`subagent` / `orchestrator`**: nested inside `reasoning:` (sibling of `reasoning.instructions` and `reasoning.actions`).

Wrong placement is a compile error.

## Authentication kind casing

Three V2 sources disagree slightly. **When in doubt, copy the canonical example.**

| Auth kind | Canonical example | V2 MCP server template (PR 1564) | V2 Beta Guide |
|---|---|---|---|
| API key | `apiKey` | `api-key` | `apiKey` |
| API key client credentials | (not used) | `api-key-client-credentials` | `apikey-client-credentials` |
| OAuth2 client credentials | `oauth2-client-credentials` | `oauth2-client-credentials` | `oauth2-client-credentials` |
| OAuth2 OBO | `oauth2-obo` | (not listed) | `oauth2-obo` |
| Basic auth | (not used) | `basic` | `basic` |
| In-task authorization code | (not used) | (not listed) | `in-task-authorization-code` |

Default: `apiKey`, `oauth2-client-credentials`, `oauth2-obo` (from canonical). For kinds the canonical doesn't use, prefer the Beta Guide's casing.

`in-task-authorization-code` is documented in the Beta Guide for OAuth2 step-up but is not in the canonical or MCP template. Don't emit unless explicitly requested.

## Variables block — the rules

```text
variables:
  response_message: string
  response_state: string
```

- Only declare variables for **cross-path shared state** (e.g., a `response_message` written by multiple paths and read by a shared echo).
- Do NOT mirror node outputs (`@<nodeType>.<nodeId>.output`) or globally-accessible request data — reference directly.
- Never declare a variable that isn't both set and read by ≥1 node.
- Syntax: `<name>: <type>` with optional indented `description:`. NO `mutable`, NO defaults like `= ""`.

## Supported LLMs — model IDs change fast

**Always look up current production model IDs at provider docs:**
- Gemini: <https://ai.google.dev/gemini-api/docs/models>
- OpenAI: <https://developers.openai.com/api/docs/models>

The canonical IT Help Network uses `gemini-2.5-flash` and `gpt-5.4`. The Beta Guide lists older OpenAI IDs (`gpt-5.2`, `gpt-5.2-pro`, `gpt-5-mini`, `gpt-5-nano`, `gpt-5`). Confirm with the user before emitting `gpt-5.4` since it's not in the published Beta Guide list. When in doubt, emit `gpt-5.2-pro` (Pro tier) or `gpt-5-mini` (mini tier).
