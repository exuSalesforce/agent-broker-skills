---
name: mulesoft-agent-broker-v1-to-v2-converter
description: Converts a MuleSoft Agent Network V1 project (schemaVersion 1.0.0) into an Agent Network V2 project (agentNetwork 2.0.0) by generating a new project folder containing a V2 agent-network.yaml, an updated exchange.json, and a brokers/<name>.agent Agent Script file with a single orchestrator node. Load this skill into context before proposing any V1→V2 changes. Use whenever the user asks to "upgrade", "convert", "migrate", "translate", or "port" an Agent Network / Agent Broker project from V1 to V2, mentions "v1 to v2", "schemaVersion 1.0.0", "agentNetwork 2.0.0", "Agent Script conversion", "broker conversion", or whenever the user is in a folder containing an Agent Network V1 YAML and wants the V2 equivalent. Also trigger when the user references Daisy planner / Agent Graph V1 → V2 migration, since "Daisy planner" and "flow" both refer to agent networks.
license: Apache-2.0
metadata:
  author: mulesoft-agent-broker-team
  version: "1.2.0"
---

# MuleSoft Agent Broker V1 to V2 Converter

This skill converts an Agent Network V1 project (schema `1.0.0`) into an Agent Network V2 project (schema `2.0.0`) by producing a brand-new project folder. The conversion is intentionally simple: every V1 broker becomes a V2 broker backed by an Agent Script `.agent` file that contains exactly one **orchestrator** node. Routing, classification, and conditional logic that V1 stuffed into a single LLM prompt remain inside that orchestrator's `system.instructions` — the upgrade does not break the prompt apart into multiple nodes.

The user is a PM working on Agent Broker (the Agent Fabric product). Do not improvise or "improve" the source instructions; preserve the user's wording. Treat the V1 file as the source of truth and only change what the V2 schema actually requires.

## When to use

Trigger this skill when:
- The user asks to convert / upgrade / migrate / port an Agent Network or Agent Broker project from V1 to V2.
- The user mentions `schemaVersion: 1.0.0` and wants `agentNetwork: "2.0.0"`.
- The current working folder contains a V1 `agent-network.yaml` and `exchange.json`.

If the user is asking general questions about V2 (without converting), point them to the docs reference instead of running the conversion.

## Workflow checklist

Always run through these steps in order. Do not skip steps.

1. **Validate the working folder is a V1 Agent Network project.** See "Step 1: Validate input".
2. **Decide where to save the converted project.** Ask the user. See "Step 2: Choose output location".
3. **Read the source files in full.** Read the V1 `agent-network.yaml` and `exchange.json` end-to-end.
4. **Translate the project.** Build the three V2 files: `agent-network.yaml`, `exchange.json`, `brokers/<broker-name>.agent`. See "Step 3: Translate".
5. **Write the new project folder** to the chosen location with the suffix `_v2_conversion`. See "Step 4: Write output".
6. **Report back** the new folder path and a short summary of what was converted.

## Step 1: Validate input

Before doing anything else, verify the user is invoking this in a folder that actually contains a V1 Agent Network project. The user's instruction is firm here — do not run the conversion against the wrong folder.

Do this:
1. Determine the user's current working folder. If unclear, ask. Treat the working folder as the V1 project root.
2. Confirm both files exist:
   - `agent-network.yaml` (or `agent-network.yml`)
   - `exchange.json`
3. Read the YAML and confirm it has `schemaVersion: 1.0.0` at the top level. This is the definitive V1 marker. (V2 uses `agentNetwork: "2.0.0"` instead.)
4. Read `exchange.json` and confirm `"classifier": "agent-network"` (V1 classifier). V2 uses `"agentic-network"`.

If any check fails — files missing, wrong schema version, wrong classifier — **stop and ask the user to point you at the correct folder**. Do not guess. Example response:

> I don't see a V1 Agent Network project in this folder. I'm looking for `agent-network.yaml` with `schemaVersion: 1.0.0` and `exchange.json` with `"classifier": "agent-network"`. What folder should I convert?

## Step 2: Choose output location

Ask the user where to save the converted project. The new folder name is the original folder name with the suffix `_v2_conversion`. For example, if the original folder is `customer-onboarding-v1`, the converted folder is `customer-onboarding-v1_v2_conversion`.

Use AskUserQuestion to offer choices like:
- Same parent directory as the source (default / recommended)
- A specific path the user provides

Do not silently overwrite an existing folder at the destination. If the target directory already exists, ask whether to overwrite or pick a different name.

## Step 3: Translate

The conversion has three outputs, all derived from the same V1 source. Read `references/v2-schema.md` for the authoritative V2 schema reference, and `references/v1-example.md` and `references/v2-example.md` for a complete worked example.

### 3a. Build the V2 `agent-network.yaml`

The V2 file has four top-level sections: `agentNetwork`, `info`, `registry`, `context`, `brokers`. Map V1 → V2 like this:

| V1 location | V2 location | Notes |
| --- | --- | --- |
| `schemaVersion: 1.0.0` | `agentNetwork: "2.0.0"` | Required string. |
| `label` (top level) | `info.label` | Required. |
| (none in V1) | `info.version` | Required. Use `1.0.0`. |
| `brokers.<id>.card.description` | `info.description` | If present, lift the broker's description into `info`. Otherwise omit. |
| `agents.<id>` | `registry.agents.<id>` | See **Agent translation** below. |
| `mcpServers.<id>` | `registry.mcps.<id>` | Note: V1 says `mcpServers`, V2 says `mcps`. |
| `llmProviders.<id>` | `registry.llms.<id>` | Move card-less LLM info under `metadata`. |
| `connections.<id>` | `context.connections.<id>` | See **Connection translation** below. |
| `brokers.<id>` (the orchestrator broker) | `brokers.<id>` + `./brokers/<id>.agent` | See **Broker translation** below. |

#### Agent translation (`agents` → `registry.agents`)

For each V1 agent:
- Lift V1's top-level `label` into `info.label`.
- Keep `metadata.protocol` and `metadata.platform` as-is.
- Build `metadata.card.a2a` using the **broker's** A2A card fields as a template (V1 leaf agents typically have no card of their own). For each leaf agent, populate the card with sensible defaults derived from V1: `name` = the agent's `label`, `description` = the agent's `label` (or a short derived sentence), `url` = `${ingressgw.url}/<agentName>`, `version: 1.0.0`, `protocolVersion: 0.3.0`, `capabilities.pushNotifications: false`, `defaultInputModes` and `defaultOutputModes` set to `[application/json, text/plain]`.
- Define one `skills` entry per agent. Use `<agentName>-<verb>` as the skill `id` (e.g. `workday-create-record`), and copy the agent's purpose into `name` and `description`.

Do not invent capabilities the V1 file didn't claim. If V1 doesn't say `pushNotifications: true`, leave it `false`.

#### MCP translation (`mcpServers` → `registry.mcps`)

For each V1 MCP server:
- Lift `label` into `info.label`. Add a short `info.description` if you can derive one safely (e.g. from the V1 broker's instructions referencing the tool).
- Keep `metadata.transport` exactly as-is.
- Add a `metadata.tools` array. For each tool the V1 broker calls (look at the broker's `tools[*].mcp.allowed` list in V1 for the canonical tool name), add an entry with `name`, `description`, and a minimal `inputSchema` (`type: object`, plus a `properties` block with the named arguments you can infer from the broker's instructions). If you genuinely cannot determine a tool's schema, leave `properties` empty rather than fabricating one.

#### LLM translation (`llmProviders` → `registry.llms`)

For each V1 LLM provider:
- Lift `label` into `info.label` and `description` into `info.description`.
- Keep `metadata.platform`.
- Add `metadata.models` as a list. Include the model the V1 broker actually uses (`spec.llm.configuration.model`) plus any obvious sibling (e.g. if V1 uses `gemini-3-flash-preview`, you can add `gemini-2.5-flash`). Do not pad the list speculatively.

#### Connection translation (`connections` → `context.connections`)

V1's `connections` section is a flat object of named connections, each with `kind`, `ref.name`, and `spec.url` (plus an optional `spec.configuration` for auth). V2's `context.connections` keeps the same idea but the URL moves up out of `spec`. Map each V1 connection like this:

```yaml
# V1
WorkdayAgentTestConnection:
  kind: agent
  ref:
    name: WorkdayAgentTest
  spec:
    url: https://...
```
becomes
```yaml
# V2
workdayAgentConnection:
  kind: a2a            # V1 'agent' becomes V2 'a2a'
  ref:
    name: workdayAgent
  url: ${workdayAgent.url}   # promote URL out of spec; parameterize via exchange.json
```

Mapping rules:
- `kind: agent` (V1) → `kind: a2a` (V2).
- `kind: mcp` and `kind: llm` stay the same.
- For LLM connections, move auth into a top-level `authentication` block: `kind: apiKey`, `apiKey: ${<llmName>.apiKey}`. Do not keep the V1 `spec.configuration` shape.
- Replace hardcoded URLs with `${<connectionRefName>.url}` variables. The actual URL value goes into `exchange.json` under `metadata.variables`.
- Connection identifiers and ref names should be camelCase in V2 (e.g. `workdayAgentConnection`, ref name `workdayAgent`). Rename V1's PascalCase identifiers consistently.

#### Broker translation (`brokers.<id>` → `brokers.<id>` + Agent Script file)

In V1, the broker definition contains both the A2A card and the orchestrator's prompt (`spec.instructions`) in one block. In V2, those are split:
- The broker's A2A card stays in `agent-network.yaml` under `brokers.<broker-id>.interfaces.a2a.card`.
- The orchestrator's prompt moves into `./brokers/<broker-id>.agent`.

Build the V2 broker entry like this:
```yaml
brokers:
  <broker-id>:
    kind: AgentScript
    implementation: ./brokers/<broker-id>.agent
    interfaces:
      a2a:
        card:
          name: <copied from V1 card.name>
          description: <copied from V1 card.description>
          url: ${ingressgw.url}/<broker-id>
          version: 1.0.0
          protocolVersion: 0.3.0
          capabilities:
            streaming: false
            pushNotifications: <copied from V1 card.capabilities.pushNotifications, default false>
          defaultInputModes: [application/json, text/plain]
          defaultOutputModes: [application/json, text/plain]
          skills:
            - id: <copied from V1 skill id, lowercased / kebab-cased>
              name: <copied from V1 skill name>
              description: <copied from V1 skill description>
              tags: <copied>
              examples: <copied if present>
              inputModes: [application/json, text/plain]
              outputModes: [application/json, text/plain]
```

Use a sensible kebab-case broker id derived from the V1 broker name (e.g. `CustomerOnboardingBrokerTest` → `customer-onboarding`).

### 3b. Build the V2 `exchange.json`

Take the V1 `exchange.json` and apply these changes:
- Change `"classifier": "agent-network"` → `"classifier": "agentic-network"`.
- Update `name`, `assetId`, and `version` to reflect the V2 conversion (e.g. add `-v2` suffix).
- Add a `description` field describing the converted network.
- For every connection that points to an external URL in the V1 YAML, add a `metadata.variables.<refName>.url` entry with the URL as `default`. This is what makes the `${<refName>.url}` references in the V2 YAML resolve.
- Preserve any existing variables (e.g. `gemini.apiKey`).
- Mark API keys as `"secret": true`. Mark URLs `"secret": false`.

### 3c. Build the Agent Script `<broker-id>.agent` file

This file contains the orchestrator node and supporting wiring. Structure:

```text
# @dialect: AGENTFABRIC=0.1-BETA

system:
  instructions: "<one-line summary of the broker's role, derived from V1 broker description>"

config:
  agent_name: "<broker-id>"
  label: "<human-readable broker label>"
  description: "<one-line description>"
  default_llm: @llm.<llm-id>

# -- LLM ----------------------------------------------------------------------
llm:
  <llm-id>:
    target: "llm://<llmConnectionName>"
    kind: "<Gemini|OpenAI>"
    model: "<model from V1 spec.llm.configuration.model>"

# -- ACTION DEFINITIONS -------------------------------------------------------
actions:
  <agent_action_id>:
    target: "a2a://<agentConnectionName>"
    kind: "a2a:send_message"
  ...
  <mcp_tool_action_id>:
    target: "mcp://<mcpConnectionName>"
    kind: "mcp:tool"
    tool_name: "<exact V1 tool name from brokers.tools[*].mcp.allowed>"
    # Do NOT add `inputs:` here — V1 has no input schemas, and the V2 runtime
    # auto-discovers tool arguments from the MCP server.

# -- TRIGGER ------------------------------------------------------------------
trigger <broker-id-camelCase>Trigger:
  kind: "a2a"
  target: "brokers://<broker-id>/a2a"
  on_message: ->
    transition to @orchestrator.<orchestratorId>

# -- ORCHESTRATOR -------------------------------------------------------------
orchestrator <orchestratorId>:
  description: "<broker description from V1>"
  label: "<broker label from V1>"
  system:
    instructions: |
      <V1 spec.instructions, copied verbatim, with one mechanical change:>
      <wherever V1 referenced a tool by full dotted name (e.g. "SlackMcpServer.send_status_update"),>
      <prefer the V2 action name (e.g. "send_slack_status_update"). Do not paraphrase or restructure>
      <the prompt; preserve the user's bullets, formatting, and intent.>
  reasoning:
    instructions: ->
      | {!@request.payload.message.parts[0].text}
    actions:
      <readable_alias>: @actions.<agent_action_id>
      ...
    outputs:
      properties:
        summary:
          type: "string"
          description: "Plain-text summary of all actions taken."
    max_number_of_loops: 25
  on_exit: ->
    transition to @echo.<responseEchoId>

# -- RESPONSE -----------------------------------------------------------------
echo <responseEchoId>:
  kind: "a2a:response"
  task: a2a.task({
    state: "completed",
    message: a2a.message({
      messageId: uuid(),
      parts: [
        a2a.textPart(@orchestrator.<orchestratorId>.output.summary)
      ]
    }),
    metadata: None
  })
```

Action naming guidance:
- For each V1 linked agent, create one `a2a:send_message` action. Convert the agent name to snake_case (e.g. `WorkdayAgentTest` → `workday_agent`).
- For each V1 MCP tool listed in the broker's `tools[*].mcp.allowed`, create one `mcp:tool` action. Pick a snake_case action id that reflects the tool's purpose (e.g. `SlackMcpServer.send_status_update` → `send_slack_status_update`). Always preserve the original `tool_name` exactly.
- **Do not add an `inputs:` block to MCP actions** when converting from V1. V1 does not declare tool input schemas — it only lists tool names in `brokers.tools[*].mcp.allowed`. The V2 MCP runtime auto-discovers a tool's arguments from the MCP server itself, and the `inputs:` field is optional (per V2 docs: *"Input arguments provided are not exhaustive. The tool will auto-discover additional arguments and consider them in slot filling mode."*). Inferring `inputs` from the orchestrator's prose prompt is fabrication and can produce a wrong schema. Leave it out and let the runtime discover.
- The only time you'd add `inputs:` is if you need to *fix* or *default* an argument value via `with` later (e.g. pinning a `channel_id`). The conversion itself shouldn't introduce that.

Reasoning action aliases:
- Inside `reasoning.actions`, use short readable aliases (e.g. `workday`, `salesforce`, `slack_update`) rather than reusing the action id verbatim. The V2 example uses this convention and it makes the orchestrator instructions more human-readable.

Prompt preservation:
- Copy `spec.instructions` from V1 verbatim into the orchestrator's `system.instructions`. Light, mechanical edits only:
  - Replace dotted MCP tool names with the new V2 action alias if the prompt names a tool.
  - Tighten obvious typos in V1 (e.g. `"this a long running task"` → `"this is a long running task"`) only if it would be confusing otherwise. When in doubt, leave the user's wording alone.
- Do not split the prompt into multiple nodes. Do not add routers, executors, or generators. The whole orchestration runs inside one orchestrator node — that is the explicit constraint of this skill.

## Step 4: Write output

Create the destination folder structure exactly:
```
<chosen_path>/<original_folder_name>_v2_conversion/
├── agent-network.yaml
├── exchange.json
└── brokers/
    └── <broker-id>.agent
```

Use `mkdir -p` for the directories and the Write tool for the files. Do not include `.mvn`, `.vscode`, or other IDE folders from the V1 source — those are not needed for the conversion. If the destination already exists, ask before overwriting.

After writing, list the created files back to the user.

## Step 5: Report

Tell the user:
- The full path to the new folder.
- The name of the broker that was converted and the orchestrator node id used.
- A one-line note that the conversion is a simple V1→V2 translation with one orchestrator node, and the user can refactor into multiple V2 nodes (router, generator, executor) later if they want stricter determinism.
- If the user wants to refactor the converted project into a richer multi-node graph (split deterministic routing into `router` nodes, move side-effects into `executor` nodes, separate cheap summary work into `generator` nodes), the `mulesoft-agent-broker-builder` skill handles that — point the user at it as the natural next step.

## Reference files

- `references/v2-schema.md` — The full Agent Network V2 Beta Guide. Read this whenever you're unsure about a V2 field name, required value, or node type.
- `references/v1-example.md` — The example V1 project (`customer-onboarding-v1`) used as the canonical input.
- `references/v2-example.md` — The example V2 project (`customer-onboarding-v2`) showing the canonical output for that input. When in doubt about formatting or naming conventions, mirror this example.

## Common mistakes to avoid

- **Don't** keep `schemaVersion: 1.0.0` — V2 uses `agentNetwork: "2.0.0"`.
- **Don't** keep the V1 `agent-network` classifier — V2 uses `agentic-network`.
- **Don't** invent extra nodes (router, generator, executor). The user explicitly wants a single orchestrator node.
- **Don't** rewrite the user's prompt to "improve" it. Preserve wording.
- **Don't** hardcode URLs in the V2 YAML — parameterize them via `${<name>.url}` and put the value in `exchange.json`.
- **Don't** forget to add an `info.version` (it's required in V2 and absent in V1).
- **Don't** put auth blocks inside the registry — auth lives on `context.connections.<name>.authentication` in V2.
- **Don't** add `inputs:` to MCP actions when converting from V1. V1 has no tool input schemas, so any `inputs:` block you write is guessed from prose. The V2 runtime auto-discovers tool arguments — leave `inputs:` off entirely.
