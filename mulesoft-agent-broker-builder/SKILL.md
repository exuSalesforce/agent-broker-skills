---
name: mulesoft-agent-broker-builder
description: Build, edit, validate, publish, and deploy a MuleSoft Agent Broker (Agent Network V2) project end-to-end. Use whenever the user asks to build, create, scaffold, configure, edit, validate, publish, or deploy an Agent Broker, agent network V2, AgentScript, `.agent` file, or `agentNetwork: 2.0.0` project. Do NOT trigger for V1→V2 migration (use the converter skill).
license: Apache-2.0
metadata:
  author: mulesoft-agent-broker-team
  version: "1.0.0"
---

# MuleSoft Agent Broker Builder

Turns a natural-language description of a multi-agent workflow into a complete, validated, deployable Agent Network V2 project. Runs the Beta Guide's 6-phase guided experience and adds publish + deploy.

Inside MuleSoft Vibes (Anypoint Code Builder), this skill replaces what `configure_agent_network_yaml` and `create_agent_network_project` would do, and calls the deterministic infra tools (`validate_project`, `publish_agent_network_assets`, `deploy_agent_network`, `search_asset`) when present. Fully portable to Claude Code, Cursor, Codex when those tools aren't available.

## When to use

- Use for: building a new Agent Broker; adding/swapping nodes, agents, MCP tools, or LLMs; tuning instructions; changing routing; running validate/publish/deploy.
- Don't use for: V1→V2 conversions (use `mulesoft-agent-broker-v1-to-v2-converter`); standalone Mule apps, DataWeave, or APIs.

If the YAML has `schemaVersion: 1.0.0`, this is V1 — stop and route to the converter. The V2 marker is `agentNetwork: 2.0.0` (unquoted, top of file).

## References (read on demand)

- **`references/canonical-example.md`** — The complete IT Help Network V2 project, sourced verbatim from `mulesoft-emu/agent-fabric-specification`. The structural template — when in doubt, copy it.
- **`references/gotchas.md`** — V2 syntax rules, compile-error rules, RULE-ASSET-MODE (inline vs Exchange), subagent-vs-orchestrator decision, auth casing, MCP tool integration, graceful degradation.

The Beta Guide is the authoritative V2 syntax reference. When it goes public, this skill will link to it. Until then, the references above cover the gotchas and worked example.

## MCP tool integration

| Tool | Behavior |
|---|---|
| `create_agent_network_project` | Replaced — skill writes scaffold directly. |
| `configure_agent_network_yaml` | Replaced — skill IS the guided experience. |
| `validate_project` | Call when present (Step 6). Falls back to structural checklist. |
| `search_asset` | Call when present (Steps 1–2). Falls back to asking the user. |
| `publish_agent_network_assets` | Call when present (Step 7). Falls back to doc link. |
| `deploy_agent_network` | Call when present (Step 8). Falls back to doc link. |

In Claude Code, MuleSoft tools appear with prefix `mcp__mulesoft__`. See `references/gotchas.md` § "MCP integration" for handshake details.

---

## Workflow A — Build a new project (Phases 1–6 + Publish/Deploy)

The 6 phases below mirror the Beta Guide Appendix exactly. Each phase ends with a stop point — wait for the user. Apply each user response immediately (no batch-then-update).

## Step 0: Pre-flight

1. **Detect existing project.** If working folder has `agent-network.yaml` + `exchange.json` + `brokers/`, ask: *"Edit this one, or scaffold new in a sibling folder?"* Default edit (Workflow B). If new, get name and continue in sibling.
2. **Detect schema version.** `schemaVersion: 1.0.0` → route to converter.
3. **Confirm intent in one sentence:** *"You want me to build a new Agent Broker that does X. Sound right?"*
4. **Capture network `info`** for `agent-network.yaml`: ask for `info.label` (required) and `info.description` (optional). Default `info.version` to `1.0.0` unless user provides one.

## Step 1: Functional Requirements (Beta Guide Phase 1 — be strict)

Make the user articulate what the broker *does* unambiguously. Do not proceed until requirements are concrete.

Capture (prompt until satisfied):

1. **Trigger:** What event starts this broker? (Almost always A2A `message/send`. Confirm.)
2. **Workflow steps and branching:** Walk me through each step. *"If X, then Y."*
3. **Hard constraints:** What is absolutely restricted? These become **router conditions**, not prompt sentences.
4. **Determinism vs open-ended:** Per step — predictable input → graph branching, or LLM judgment → LLM-powered node.
5. If user gives a vague answer twice in a row, accept as placeholder, flag, move on.

Stop. Summarize and confirm: *"Here's what I have: [...]. Anything missing?"* Stop.

After confirmation, **identify required assets**:
- **LLMs** (always at least one).
- **Tools** (external capabilities — list capabilities; choice of MCP vs A2A happens in Phase 2).

Present: *"Based on requirements, I've identified these asset needs: [list]. Confirm or adjust?"* Stop.

If `search_asset` is available, **preview Exchange assets** for each capability with `assetFilters: ["llm"]`, `["mcp"]`, `["agent"]`. Group results: *"Found in Exchange: [...]. Not found: [...]."* Save preview for Phase 2. Preview only — no file writes yet.

## Step 2: Asset Registration (Beta Guide Phase 2)

Register every required asset in `agent-network.yaml` and `exchange.json`. Sub-steps run sequentially. After each individual asset, ask "Add another?" before continuing.

**Each asset is in one of two modes** — see `references/gotchas.md` § "RULE-ASSET-MODE" for the full split. Short version:
- **Inline** (asset NOT in Exchange): registry entry + `context.connections` with `ref.name`.
- **Exchange** (asset already published): `exchange.json.dependencies` + `context.connections` with `ref.name` AND `ref.namespace`. **No registry entry.**

Auth always lives on `context.connections.<id>.authentication`. Parameterize URLs/secrets via `${<name>.url}` / `${<name>.apiKey}`. Add corresponding `exchange.json.metadata.variables`. Mark secrets `"secret": true`.

### 2a — LLM(s) (mandatory)

Inline schema: `info.label` required, `metadata.platform: Gemini | OpenAI | AzureOpenai`. Connection has `kind: llm` + `url` + `authentication`. Ask "Add another?"

### 2b — MCP server(s) (conditional)

Inline schema requires `info.label` and `metadata.transport.kind` (`streamableHttp` | `sse` | `stdio`).

After registering: ask the user **what `tool_name`** they intend to call (required for action definition). Ask if they know the input parameters. If yes, declare in action `inputs:`. If no, omit `inputs:` and the runtime auto-discovers. **Never fabricate tool names or input schemas.** Ask "Add another?"

### 2c — A2A agent(s) (conditional)

Inline schema requires `info.label`, `metadata.protocol: a2a`, `metadata.card.a2a` (full A2A Agent Card with skills). Ask "Add another?"

## Step 3: Define the Broker (Beta Guide Phase 3 — Agent Script)

Write the structural skeleton of the `.agent` file. **Brief drafts** of `system.instructions` and `prompt`/`reasoning.instructions` only — Phase 5 refines prose.

Decisions:

1. **`agent_name`** — kebab-case (e.g. `it-help-investigation`). Also the `.agent` filename stem and broker id.
2. **Trigger.** Each broker has exactly one trigger per declared interface. `kind: "a2a"`, `target: "brokers://<broker-id>/a2a"`, `on_message:` is a fixed `transition to` (conditionals here are a compile error — use a router).
3. **Node decomposition.** Per workflow step:
   - Deterministic routing on a known value? → `router`.
   - Fixed side-effect (run one tool with known args, set a variable)? → `executor`.
   - Open-ended judgment, NO actions, NO HITL? → `generator`.
   - Open-ended judgment with actions OR HITL? → `subagent` (default LLM-powered type).
   - Coordinating MULTIPLE actions toward a compound goal? → `orchestrator`.
   - Final response to A2A client? → `echo` with `kind: "a2a:response"`.
4. **`subagent` is the default.** Use `orchestrator` ONLY for compound multi-action coordination. Single-action (even with A2A) → `subagent`.
5. **HITL is built into `subagent`.** When a subagent needs user input, the runtime auto-sends `input-required`. Do NOT model clarification as `subagent → router → executor → echo` — anti-pattern.
6. **Hard constraints from Phase 1 → router nodes, not prompts.** The constrained action goes in an `executor` gated by a `router`. Prompts can be ignored, router conditions cannot.
7. **First node after trigger should NOT be a pass-through executor** that copies `@request.payload.message`. Reference `@request.payload.message.parts[0].text` directly downstream.
8. **Edges.** Every non-terminal node has `on_exit: -> transition to @<nodeType>.<nodeId>`. Routers declare `routes:` + `otherwise:` and have NO `on_exit` (`transition to` inside router `on_exit` is a compile error).
9. **Echo terminus.** Every reachable path ends in `echo`. `a2a.task(...)`, `a2a.message(...)`, `a2a.textPart(...)` are functions, NOT references — don't prefix with `@`. `state` must be: `submitted | working | input-required | completed | failed | canceled | rejected`.

For full compile-error rules see `references/gotchas.md` § "Compile-error rules". For the structural template see `references/canonical-example.md`.

Update the `brokers` entry in `agent-network.yaml` to reference the new `.agent` file.

## Step 4: Asset Assignment to Graph (Beta Guide Phase 4)

Bind actions to nodes, respecting least-privilege.

1. **Action definitions** in the `.agent` file:
   - **A2A**: `target: "a2a://<connection>"`, `kind: "a2a:send_message"`. **No `inputs:`.**
   - **MCP**: `target: "mcp://<connection>"`, `kind: "mcp:tool"`, `tool_name: "..."`, optional `inputs:`. Only declare `inputs:` if you know the parameters.

2. **Action invocation rules** — see `references/gotchas.md` § "Compile-error rules" for the full table. Critical: A2A inside `reasoning.actions` is bare reference (no `with message =`); A2A inside `executor` `do: run` REQUIRES `with message =`.

3. **`http_headers` is the only implicit parameter** — never declare in `inputs:`. Use to propagate auth: `with http_headers = {"Authorization": @request.headers["Authorization"]}`.

4. **Asset cap (guideline)** — ~4 actions max per `subagent`/`orchestrator`. Beyond that, propose splitting or moving deterministic actions to `executor`.

5. **LLM choice per node.** Reasoning-heavy → Pro tier. Cheap classification/summary → Flash/mini tier. Use `default_llm` for common case; override per-node only on exceptions.

6. **Action aliases** inside `reasoning.actions` — short readable names (`search_help`, `slack_update`). The alias is what the LLM sees.

**Least-privilege (CR-18 nuance).** Irreversible/high-stakes mutations (escalate, delete, send-to-customer) MUST be in `executor` nodes gated by `router`. Idempotent updates (status updates, ticket notes) MAY live on a `subagent`/`orchestrator`. The line: irreversible (executor-only, router-gated) vs idempotent/domain-natural (orchestrator OK).

## Step 5: Instruction Refinement (Beta Guide Phase 5)

Make every LLM-powered node's prompt precise, testable, non-conflicting. **One node at a time, never batch.**

For each `generator`/`subagent`/`orchestrator`:

1. State which node: *"Refining `classifySeverity` (generator)."*
2. Show V1 prompt (brief draft from Phase 3).
3. Ask: *"What's your overall feedback? Any few-shot examples?"* Stop.
4. **Write V2:** rewrite as a numbered routine. Each step → one tool call or output decision. Avoid "be helpful"/"be professional".
5. **Make outputs concrete:** if the node has `outputs.properties.severity.enum`, the prompt must say *"Set severity to high when X, low otherwise"* matching the enum exactly.
6. **Anticipate edge cases:** what if a key field is missing? Bake the answer in.
7. **Per-node contradiction test:** does any prompt step get ruled out by deterministic graph?
8. Apply V2 to file immediately. Move to next node.

After all nodes refined: **cross-node contradiction test** — does any prompt conflict with another? Resolve with user.

## Step 6: Final Topology Review (Beta Guide Phase 6)

Cleanup, then validate.

**Cleanup:**
1. Remove unresolved `{{...}}` markers, unused `exchange.json` variables.
2. Remove unused `registry` assets (not referenced by `context.connections`, actions, or LLM blocks).
3. Remove unused `context.connections` entries.
4. Remove orphaned action definitions in `.agent` file.
5. Remove empty YAML keys (`registry.llms:` with no entries → drop the key).

**Validate:**
- If `validate_project` is available, call it with `projectPath`. Otherwise run the structural checklist below and tell the user to run the runtime validator separately.

**Structural checklist:**
- Every node reachable from trigger? No orphans.
- Every reachable path terminates at `echo`?
- Every `router` has `routes` (≥1) and `otherwise`?
- Every `subagent`/`orchestrator` has `reasoning.instructions` AND ≥1 action OR HITL?
- Every action referenced in nodes matches an entry in `actions:`?
- Every `${...}` variable has matching `exchange.json.metadata.variables` entry?
- Every `context.connections.<id>.ref.name` resolves to a registry entry OR `exchange.json.dependencies`?
- All A2A actions in `reasoning.actions` are bare references (no `with message =`)?
- All MCP `with` parameters reference declared `inputs:` fields?
- All echo `state` values in the allowed enum?
- Hard constraints enforced via router conditions, not prompts?
- Operational caps set: `max_number_of_loops` lowered from default 25 (e.g., 3 classification, 5–10 orchestration); `task_timeout_secs` set if calling slow downstream agents?

**Fix loop:**
- Group errors by file. Auto-fix formatting/indentation/missing required keys. For schema-required references missing, ask the user — **never auto-insert missing assets**. Loop until clean.

When clean: *"Project is validated and ready. Want to publish to Exchange and deploy to runtime?"*

## Step 7: Publish (optional)

Only if user wants to.

Ask `assetVersion` (semver). Optional: `groupId` (defaults to active org), `projectPath`.

If `publish_agent_network_assets` is available, call it. Surface published asset URLs.

If not available, point at: <https://docs.mulesoft.com/anypoint-code-builder/af-publish-agent-network-assets>. Don't paste CLI commands.

## Step 8: Deploy (optional)

Only if user wants to. Publish must have happened first.

Required: a Managed Omni Gateway, filled-in `exchange.json` secrets, deployment target details.

Ask: `environmentName`, `privateSpaceName`, `ingressGatewayName`, `egressGatewayName`.

If `deploy_agent_network` is available, call it. Surface deployment URL/ID.

If not available, point at: <https://docs.mulesoft.com/anypoint-code-builder/af-deploy-agent-network-targets>.

---

## Workflow B — Edit an existing project

**Universal edit principles:**

- **Impact analysis first.** Identify every downstream node, expression, or connection that depends on the change. Confirm with user.
- **Minimal disruption.** Edit only what's asked. Exception: node-type changes (subagent ↔ orchestrator) when actions are added/removed.
- **Registry-first.** New assets go in `agent-network.yaml` registry/context (or `exchange.json.dependencies`) BEFORE being referenced from `.agent`.
- **Defer validation while editing.** Lint errors mid-edit are expected; validate at the end.

### B.1 — Add or swap an asset

1. Decide mode (Inline or Exchange — see `gotchas.md`).
2. **Inline:** `registry.{agents,mcps,llms}` + `context.connections.<id>` with auth + `exchange.json.metadata.variables`. **Exchange:** `exchange.json.dependencies` + `context.connections.<id>` with `ref.name` + `ref.namespace`.
3. Add `actions:` entry in the `.agent` file.
4. Reference from target node's `reasoning.actions` (subagent/orchestrator) or `do: run` (executor).
5. **Re-evaluate node type if action count changed.** Single action → `subagent`. Multi-action → `orchestrator`. Update `@<oldType>.<id>` references downstream.
6. **Cap check.** If a node now exceeds 4 actions, propose splitting.
7. **Update `system.instructions`** to explain how to use the new asset.
8. Run Step 6 (validate).

### B.2 — Change graph logic

Edit `.agent`. After any change, walk trigger → every echo and confirm:
- Every router `when:` references a real, reachable upstream output (not `@request.payload` directly).
- Hard constraints still in router conditions, not prompts.
- No node became unreachable or dead-end.
- No `transition to` in a router's `on_exit` (compile error).

Run Step 6.

### B.3 — Refine instructions

Edit `.agent` only. Apply Phase 5 contradiction tests. If user pastes a policy doc, transform into numbered routine — don't paste verbatim. Run Step 6.

### B.4 — Add or change a connection policy

Edit `agent-network.yaml`. V2 supports policies even though the configure-instructions template doesn't surface them in build flow.

A policy has **two parts**:
1. **Definition** — `context.policies.<policyId>`.
2. **Binding** — referenced from `context.connections.<connId>.policies.outbound` or `brokers.<brokerId>.interfaces.a2a.policies.{inbound,outbound}`. Always reference by id.

For Exchange-published policies: add to `exchange.json.dependencies` first, then bind. Run Step 6.
