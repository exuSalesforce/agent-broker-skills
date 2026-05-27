---
name: mulesoft-agent-broker-builder
description: Build, configure, edit, validate, publish, and deploy a MuleSoft Agent Broker (Agent Network V2) project end-to-end through the Beta Guide's 6-phase guided experience. Produces a complete project with `agent-network.yaml`, `exchange.json`, and one or more `brokers/<name>.agent` Agent Script files. Also handles editing an existing project (asset swaps, graph logic changes, instruction refinement, policy tuning). Load this skill into context before proposing any build, edit, validate, publish, or deploy action on a V2 project. Use whenever the user asks to "build", "create", "scaffold", "generate", "set up", "design", "configure", "validate", "publish", "deploy", or "edit" an Agent Broker, Agent Network V2, agent graph V2, Agent Fabric flow, or AgentScript project; mentions terms like `agentNetwork: "2.0.0"`, `.agent` file, broker, orchestrator node, A2A trigger, MCP server registration, "brokers folder", or `validate_project`; asks to add or swap an A2A agent / MCP tool / LLM in an existing broker; asks to change routing or branching of a broker's graph; wants to tighten the system instructions for an LLM-powered node; or wants to add or change a connection policy. Trigger even when the user doesn't say "V2" by name — terms like "agent broker", "build me an orchestrator", "publish my agent broker", or "create a broker that does X" all imply this skill. Do NOT trigger for "Daisy planner" or V1→V2 migration — that is the V1→V2 converter skill's domain.
license: Apache-2.0
metadata:
  author: mulesoft-agent-broker-team
  version: "3.0.0"
---

# MuleSoft Agent Broker Builder

This skill turns a natural-language description of a multi-agent workflow into a complete, validated, deployable Agent Network V2 (Agent Broker) project. It runs the Beta Guide's 6-phase guided experience and adds publish + deploy lifecycle.

The skill is **the V2 guided experience.** Inside MuleSoft Vibes (Anypoint Code Builder), it replaces what `configure_agent_network_yaml` and `create_agent_network_project` would do, calls the deterministic infra tools (`validate_project`, `publish_agent_network_assets`, `deploy_agent_network`) when present, and is fully portable to Claude Code, Cursor, Codex, etc. when those tools aren't available.

## Source of truth

**The Agent Broker (Agent Script) Beta Guide is the authoritative reference for V2 syntax.** This skill leans on it; the references in this directory only cover gotchas, opinionated guidance, and the canonical worked example.

When the Beta Guide goes public, the skill will link to it directly. Today, see `references/v2-schema.md` in the V1→V2 converter skill (`/Users/e.xu/.claude/skills/mulesoft-agent-broker-v1-to-v2-converter/references/v2-schema.md`) for the latest copy.

## When to use vs. when not to use

- **Use this skill** for: building a new Agent Broker project; adding/removing/swapping nodes, agents, MCP tools, or LLMs; tuning system instructions; changing routing logic; running `validate_project`; publishing assets; deploying brokers.
- **Don't use this skill** for: V1 → V2 conversions (use `mulesoft-agent-broker-v1-to-v2-converter` instead); writing standalone Mule applications, DataWeave, or APIs.

If the YAML has `schemaVersion: 1.0.0`, this is V1. Stop and tell the user to run the V1→V2 converter skill first. The V2 marker is `agentNetwork: "2.0.0"` (quoted, top of file).

## How this skill relates to the MuleSoft MCP server

| MCP tool | Skill behavior |
|---|---|
| `create_agent_network_project` | **Replaced.** Skill writes scaffold files directly. |
| `configure_agent_network_yaml` | **Replaced.** Skill *is* the guided experience. |
| `validate_project` | **Calls when present.** Falls back to a structural checklist. |
| `publish_agent_network_assets` | **Calls when present.** Falls back to surfacing the Anypoint CLI doc link. |
| `deploy_agent_network` | **Calls when present.** Falls back to surfacing the Anypoint CLI doc link. |
| `search_asset` | **Calls when present.** Falls back to asking the user for asset URLs/IDs. |

See `references/mcp-integration.md` for the exact handshake.

## Reference files (read on demand)

- **`references/canonical-example.md`** — The complete IT Help Network V2 project, sourced verbatim from `mulesoft-emu/agent-fabric-specification`. The structural template — when in doubt, copy it.
- **`references/agent-script-grammar.md`** — V2 syntax gotchas the Beta Guide doesn't make obvious: compile-error rules, echo state enum, auth kind casing table, dialect line, subagent-vs-orchestrator decision.
- **`references/asset-selection.md`** — Inline-vs-Exchange asset registration (RULE-ASSET-MODE), asset cap, least-privilege rules, the subagent-vs-orchestrator decision tree.
- **`references/mcp-integration.md`** — How to detect and call MuleSoft MCP server tools and degrade gracefully when not present.

---

## Workflow A — Build a new project (Phases 1–6 + Publish/Deploy)

The 6 phases below mirror the Beta Guide Appendix exactly. Each phase ends with a stop point — wait for the user before continuing. Apply each user response to files immediately (no batch-then-update).

## Step 0: Pre-flight

Before starting Phase 1:

1. **Detect existing project.** If the working folder has `agent-network.yaml` + `exchange.json` + `brokers/`, ask: *"Edit this one, or scaffold a new project in a sibling folder?"* Default to edit (Workflow B). If new, get the project name and continue in a sibling folder.
2. **Detect schema version.** If `schemaVersion: 1.0.0`, stop and route to the V1→V2 converter.
3. **Confirm intent in one sentence:** *"You want me to build a new Agent Broker project that does X. Sound right?"*

## Step 1: Functional Requirements (Beta Guide Phase 1 — be strict)

Make the user articulate what the broker *does* in unambiguous terms. Do not proceed until requirements are concrete.

Capture (prompt until satisfied):

1. **Trigger:** What event starts this broker? (Almost always an A2A `message/send`. Confirm.)
2. **Workflow steps and branching:** Walk me through each step. *"If X, then Y"* form.
3. **Hard constraints:** What is absolutely restricted? (Examples: never escalate without a Jira ID; never restart Service X without approval.) These become **router conditions**, not prompt sentences.
4. **Determinism vs. open-endedness:** For each step:
   - **Deterministic** (predictable from input → graph branching).
   - **Open-ended** (requires LLM judgment → LLM-powered node).
5. If the user gives a vague answer, ask follow-ups. If they give a vague answer twice in a row, accept as a placeholder, flag, and move on.

Stop. Then summarize and confirm: *"Here's what I have: [...]. Anything missing?"* Stop again.

After confirmation, **identify required assets**:
- **LLMs** (always at least one).
- **Tools** (external capabilities — list capabilities without categorizing as MCP or A2A; choice happens in Phase 2).

Present: *"Based on requirements, I've identified these asset needs: [list]. Confirm or adjust?"* Stop.

If `search_asset` is available, **preview Exchange assets** for each capability with `assetFilters: ["llm"]`, `["mcp"]`, `["agent"]`. Group results: *"Found in Exchange: [...]. Not found: [...]."* Save preview for Phase 2. Preview only — no file writes yet.

## Step 2: Asset Registration (Beta Guide Phase 2)

Register every required asset in `agent-network.yaml` and `exchange.json`. Sub-steps 2a, 2b, 2c run sequentially. After each individual asset, ask "Add another?" before continuing.

**The two asset registration modes — RULE-ASSET-MODE.** Each asset is in **one of two modes**:

1. **Inline** (asset NOT in Exchange): Update **two places**: `registry.{agents,mcps,llms}.<id>` definition + `context.connections.<connId>` with `ref.name: <id>` matching the registry id.
2. **Exchange** (asset already published): Update **two places**: `exchange.json.dependencies` entry + `context.connections.<connId>` with `ref.name: <assetId>` AND `ref.namespace: <groupId>`. **No registry entry.**

Each asset chooses its mode independently. A single project can mix modes.

Auth always lives on `context.connections.<id>.authentication`, never inside `registry`. Parameterize URLs/secrets via `${<name>.url}` / `${<name>.apiKey}`. Add corresponding entries to `exchange.json.metadata.variables`. Mark secrets `"secret": true`.

For supported auth kinds, schemas, and the Exchange-vs-inline pattern, read `references/asset-selection.md` and `references/agent-script-grammar.md`.

### 2a — LLM(s) (mandatory, always at least one)

For each LLM the user picks, register inline or as Exchange dependency. Inline schema: `info.label` required, `metadata.platform: Gemini | OpenAI | AzureOpenai`. Add `context.connections.<id>` with `kind: llm` + `url` + `authentication`.

Ask "Add another LLM?"

### 2b — MCP server(s) (conditional)

If no tool needs map to MCP, ask: *"Want to add any MCP servers anyway?"* If no, skip to 2c.

For each MCP: inline schema requires `info.label` and `metadata.transport.kind` (`streamableHttp` | `sse` | `stdio`). Optional `metadata.tools[]`.

After registering: ask the user **what `tool_name`** they intend to call (required for action definition in `.agent`). Ask if they know the input parameters. If yes, declare in the action's `inputs:`. If no, omit `inputs:` and the runtime auto-discovers. **Never fabricate tool names or input schemas.**

Ask "Add another MCP?"

### 2c — A2A agent(s) (conditional)

If no A2A delegation needed, ask: *"Want to add any external A2A agents anyway?"* If no, skip to Phase 3.

For each agent: inline schema requires `info.label`, `metadata.protocol: a2a`, `metadata.card.a2a` (full A2A Agent Card with skills).

Ask "Add another agent?"

## Step 3: Define the Broker (Beta Guide Phase 3 — Agent Script)

Write the structural skeleton of the `.agent` file. **Brief drafts** of `system.instructions` and `prompt`/`reasoning.instructions` only — Phase 5 refines prose.

Decisions:

1. **`agent_name`** — kebab-case, derived from purpose (e.g. `it-help-investigation`). Also the `.agent` filename stem and the broker id in `agent-network.yaml`.
2. **Trigger.** Each broker has **exactly one** trigger per declared interface. `kind: "a2a"`, `target: "brokers://<broker-id>/a2a"`, `on_message:` is a fixed `transition to` (conditionals here are a compile error — use a router).
3. **Node decomposition.** For each workflow step:
   - Deterministic routing on a known value? → `router`.
   - Fixed side-effect (run one tool with known args, set a variable)? → `executor`.
   - Open-ended judgment with NO actions and NO HITL? → `generator`.
   - Open-ended judgment with actions OR human-in-the-loop? → `subagent` (default LLM-powered node type).
   - Coordinating MULTIPLE actions toward a compound goal? → `orchestrator`.
   - Final response to the A2A client? → `echo` with `kind: "a2a:response"`.
4. **`subagent` vs `orchestrator`** (the key call): `subagent` is the **default**. Use `orchestrator` ONLY for compound multi-action coordination (e.g., "call HR agent → send Slack notification → update Jira"). Single-action even with A2A → still `subagent`.
5. **HITL is built into `subagent`.** When a subagent needs user input, the runtime auto-sends an `input-required` A2A response. Do NOT model clarification as `subagent → router on needs_clarification → executor → echo` — that's an anti-pattern. Just let the subagent ask.
6. **Hard constraints from Phase 1 → router nodes, not prompts.** The constrained action goes in an `executor` gated by a `router`. Never let an LLM-powered node carry the constrained action — prompts can be ignored, router conditions cannot.
7. **First node after trigger should NOT be a pass-through executor** that only copies `@request.payload.message` into a variable. Reference `@request.payload.message.parts[0].text` directly downstream.
8. **Edges.** Every non-terminal node has `on_exit: -> transition to @<nodeType>.<nodeId>`. Routers declare `routes:` + `otherwise:` and have NO `on_exit` (using `transition to` inside a router's `on_exit` is a compile error).
9. **Echo terminus.** Every reachable path ends in an `echo` node. `a2a.task(...)`, `a2a.message(...)`, `a2a.textPart(...)` are functions, NOT references — don't prefix with `@`. `state` must be one of: `submitted`, `working`, `input-required`, `completed`, `failed`, `canceled`, `rejected`.

Read `references/agent-script-grammar.md` for the compile-error rules and `references/canonical-example.md` for the structural template.

Update the `brokers` entry in `agent-network.yaml` to reference the new `.agent` file.

## Step 4: Asset Assignment to Graph (Beta Guide Phase 4)

Bind actions to nodes, respecting least-privilege.

1. **Action definitions in the `.agent` file:**
   - **A2A action**: `target: "a2a://<connectionName>"`, `kind: "a2a:send_message"`. **No `inputs:`.**
   - **MCP action**: `target: "mcp://<connectionName>"`, `kind: "mcp:tool"`, `tool_name: "<exact name>"`, optional `inputs:`. Only declare `inputs:` if you actually know the parameters.

2. **Action invocation rules — compile errors when wrong:**
   - **Inside `subagent`/`orchestrator` `reasoning.actions`:**
     - **A2A actions are bare references.** No `with message =`. The LLM picks the message at runtime.
     - **MCP actions can have `with` clauses** for any parameter declared in `inputs:`. Slot-fill `with <param> = ...` (literal ellipsis) is allowed. If `inputs:` is omitted, the invocation MUST have zero `with` parameters (other than `http_headers`).
   - **Inside `executor` `do:` `run` statements:**
     - **A2A actions REQUIRE `with message = <value>`** where `<value>` is a string literal, `@reference`, or concatenation.
     - **MCP**: same `inputs`-must-be-declared rule. No slot-fill.

3. **`http_headers` is the ONLY implicit parameter** on every action — never needs declaration. Use to propagate auth: `with http_headers = {"Authorization": @request.headers["Authorization"]}`.

4. **Asset cap (guideline).** ~4 actions max per `subagent`/`orchestrator`. Beyond that, propose splitting or moving deterministic actions to `executor`. Don't refuse if the user accepts the risk.

5. **LLM choice per node.** Reasoning-heavy → Pro tier. Cheap classification/summary/format → Flash/mini tier. Use `default_llm` for the common case; override per-node only on exceptions.

6. **Action aliases inside `reasoning.actions`** — short readable names (`search_help`, `slack_update`). The alias is what the LLM sees.

For the full asset assignment rules, read `references/asset-selection.md`.

## Step 5: Instruction Refinement (Beta Guide Phase 5)

Make every LLM-powered node's prompt precise, testable, non-conflicting. One node at a time, never batch.

For each `generator`/`subagent`/`orchestrator`:

1. State which node: *"Refining `classifySeverity` (generator)."*
2. Show the V1 prompt (brief draft from Phase 3).
3. **Ask for feedback and success criteria:** *"What's your overall feedback? Any few-shot examples to incorporate?"* Stop.
4. **Write V2:** rewrite as a numbered routine. Each step → one tool call or output decision. Avoid "be helpful"/"be professional".
5. **Make outputs concrete:** if the node has `outputs.properties.severity.enum`, the prompt must say *"Set severity to high when X, low otherwise"* matching the enum exactly.
6. **Anticipate edge cases:** what if a key field is missing? Bake the answer into the prompt.
7. **Per-node contradiction test:** does any step in this prompt get ruled out by the deterministic graph?
8. Apply V2 to the file immediately. Move to next node.

After all nodes refined: **cross-node contradiction test** — does any prompt conflict with another? Resolve with the user.

## Step 6: Final Topology Review (Beta Guide Phase 6)

Cleanup, then validate.

**Cleanup:**
1. Remove unresolved `{{...}}` markers, unused `exchange.json` variables.
2. Remove unused `registry` assets (not referenced by `context.connections`, actions, or LLM blocks).
3. Remove unused `context.connections` entries.
4. Remove orphaned action definitions in the `.agent` file.
5. Remove empty YAML keys (`registry.llms:` with no entries → drop the key).

**Validate:**
- If `validate_project` is available, call it with `projectPath`. Otherwise run the structural checklist below by hand and tell the user to run the runtime validator separately.

**Structural checklist (always):**
- Every node reachable from trigger? No orphans.
- Every reachable path terminates at an `echo`?
- Every `router` has both `routes` (≥1) and `otherwise`?
- Every `subagent`/`orchestrator` has `reasoning.instructions` AND ≥1 action OR HITL pattern?
- Every action referenced in nodes matches an entry in `actions:`?
- Every `${...}` variable in YAML has matching entry in `exchange.json.metadata.variables`?
- Every `context.connections.<id>.ref.name` resolves to a registry entry OR an `exchange.json.dependencies` entry?
- All A2A actions in `reasoning.actions` are bare references (no `with message =`)?
- All MCP `with` parameters reference declared `inputs:` fields?
- All echo `state` values are in the allowed enum?
- Hard constraints from Phase 1 enforced via router conditions, not prompts?
- Operational caps set: `max_number_of_loops` lowered from default 25 for tight loops (e.g., 3 for classification, 5–10 for orchestration); `task_timeout_secs` set if calling slow downstream agents?

**Fix loop:**
- Group reported errors by file. For each:
  - Formatting/indentation/missing required keys → fix automatically, re-run.
  - Schema-required references missing → ask the user to fix or remove. **Never auto-insert missing assets.**
- Loop until clean.

When clean: *"Project is validated and ready. Want to publish to Exchange and deploy to runtime?"*

## Step 7: Publish (optional)

Only if user wants to publish.

Ask for `assetVersion` (semver). Optional: `groupId` (defaults to active org), `projectPath`.

If `publish_agent_network_assets` is available, call it. Surface the published asset URLs.

If not available, point at the doc: <https://docs.mulesoft.com/anypoint-code-builder/af-publish-agent-network-assets>. Don't paste CLI commands — flag spelling drifts.

## Step 8: Deploy (optional)

Only if user wants to deploy. Publish must have happened first (or assets already in Exchange).

Required: a Managed Omni Gateway, filled-in `exchange.json` secrets, deployment target details.

Ask: `environmentName`, `privateSpaceName`, `ingressGatewayName`, `egressGatewayName`.

If `deploy_agent_network` is available, call it. Surface the deployment URL/ID.

If not available, point at: <https://docs.mulesoft.com/anypoint-code-builder/af-deploy-agent-network-targets>.

---

## Workflow B — Edit an existing project

**Universal edit principles:**

- **Impact analysis first.** Before changing anything, identify every downstream node, expression, or connection that depends on it. Confirm with the user.
- **Minimal disruption.** Edit only what's asked. Exception: node-type changes (subagent ↔ orchestrator) when actions are added/removed.
- **Registry-first.** New assets go in `agent-network.yaml` registry/context (or `exchange.json.dependencies`) BEFORE being referenced from `.agent` files.
- **Defer validation while editing.** Lint errors mid-edit are expected; validate at the end.

### B.1 — Add or swap an asset (tool / agent / LLM)

1. Decide mode: Inline or Exchange (RULE-ASSET-MODE).
2. **Inline:** add to `registry.{agents,mcps,llms}` + `context.connections.<id>` with auth + `exchange.json.metadata.variables` for any new `${...}` references.
   **Exchange:** add to `exchange.json.dependencies` + `context.connections.<id>` with `ref.name` + `ref.namespace`. No registry entry.
3. Add an `actions:` entry in the relevant `.agent` file.
4. Reference it from the target node's `reasoning.actions` (subagent/orchestrator) or `do:` `run` (executor).
5. **Re-evaluate node type if action count changed.** No actions → previously a `generator` should still be one (or convert if HITL needed). Single action → `subagent`. Multi-action → `orchestrator`. Update `@<oldType>.<id>` references downstream.
6. **Cap check.** If a node now exceeds 4 actions, propose splitting.
7. **Update the node's `system.instructions`** to explain how to use the new asset.
8. Run Step 6 (validate) when done.

### B.2 — Change graph logic (routes, conditions, variables)

Edit the `.agent` file. After any change, walk trigger → every echo and confirm:
- Every router `when:` references a real, reachable upstream output (not `@request.payload` directly).
- Hard constraints still in router conditions, not prompts.
- No node became unreachable or a dead-end.
- No `transition to` in a router's `on_exit` (compile error).

Run Step 6 when done.

### B.3 — Refine instructions

Edit the `.agent` file only. Apply Phase 5 contradiction tests. If the user pastes a policy doc, transform into a numbered routine — don't paste verbatim.

Run Step 6 when done.

### B.4 — Add or change a connection policy

Edit `agent-network.yaml`. V2 supports policies even though the configure-instructions template doesn't surface them in the build flow — they're valid additions during edit.

A policy has **two parts** that must stay in sync:

1. **Definition** — `context.policies.<policyId>` describes the policy and its configuration.
2. **Binding** — referenced from either `context.connections.<connId>.policies.outbound` or `brokers.<brokerId>.interfaces.a2a.policies.{inbound,outbound}`. Always reference by id; never duplicate inline.

For Exchange-published policies: add to `exchange.json.dependencies` first, then bind.

Run Step 6 when done.

---

## Common mistakes to avoid

- **Don't** put `authentication:` blocks inside `registry`. Auth lives on `context.connections.<name>.authentication`.
- **Don't** hardcode URLs or secrets in YAML. Parameterize via `${<name>.url}` / `${<name>.apiKey}`.
- **Don't** fabricate MCP `tool_name` values or `inputs:` schemas.
- **Don't** use `orchestrator` for a single-action node — use `subagent`.
- **Don't** model HITL as an external `subagent → router → executor` flow. Subagents have built-in HITL.
- **Don't** put hard constraints in prompts. Use router conditions on identifying attributes (service name + recommended action), not derived judgment flags (`requires_approval`).
- **Don't** put irreversible mutations (escalate, delete, send-to-customer) on `subagent`/`orchestrator`. They go in `executor` gated by router.
- **Don't** prefix `a2a.task` / `a2a.message` / `uuid()` with `@` — they're functions, not references.
- **Don't** attach `with message = ...` to A2A actions inside `reasoning.actions`. Bare references only there.
- **Don't** add `with` parameters not declared in the action's `inputs:` — compile error.
- **Don't** use `{!@...}` template syntax inside `a2a.*()` echo helpers. Use direct `@reference` or concatenation.
- **Don't** auto-insert missing assets during validation. Ask the user to fix or remove.
- **Don't** skip Phase 5 (instruction refinement). The graph can be perfect and still fail because two prompts contradict.
