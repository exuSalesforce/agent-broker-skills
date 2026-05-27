# MuleSoft MCP Server Integration

The MuleSoft MCP server is auto-installed in Anypoint Code Builder (and therefore in MuleSoft Vibes). It exposes 40+ tools across MuleSoft surfaces. This skill cares about a focused subset.

This file tells you which tools to call, when, how, and how to degrade gracefully when they aren't available (the user is in standalone Claude Code / Cursor / Codex without the MCP server installed).

## Tool inventory — Agent Network workflows

| Tool | Skill behavior | Why |
|---|---|---|
| `create_agent_network_project` | **Skip.** Skill writes scaffold files directly using the canonical example as template. | Pure plumbing. Calling this MCP tool just to scaffold means the skill then has to re-write files. Wasted work. |
| `configure_agent_network_yaml` | **Skip.** Skill *is* the guided experience. | This MCP tool returns a 429-line Markdown prompt template for the calling LLM to follow. The skill replaces that prompt with portable, agent-agnostic logic. Calling it would inject a redundant (and possibly contradictory) set of instructions into context. |
| `validate_project` | **Call when present.** | The runtime's own validator. Catches schema errors, missing references, expression validity. The skill's structural checklist is a subset of what this can check. |
| `search_asset` | **Call when present.** | Searches Anypoint Exchange for existing assets in Step 2 (preview) and Steps 3a–3c (registration). Materially better than asking the user for IDs by hand. |
| `publish_agent_network_assets` | **Call when present** (Step 9). | Needs Anypoint auth context. Skill cannot replicate. |
| `deploy_agent_network` | **Call when present** (Step 10). | Same — needs auth context. |
| `open_project` | Optional — call after build to open the project in the IDE pane. | – |
| `test_connection` | Optional — verify a configured connection works before deploying. | – |

## Detection

In Claude Code, MuleSoft's tools appear with the prefix `mcp__mulesoft__` (e.g., `mcp__mulesoft__search_asset`). If you see them in your tool list, the server is connected. Different hosts may use different prefixes; the semantic name (`search_asset`, `validate_project`, etc.) is stable.

```
if a tool named like search_asset or mcp__mulesoft__search_asset is exposed:
    use the MCP path
else:
    use the fallback path described below
```

Don't probe via env vars or filesystem — let the host abstract that.

---

## How to call each tool

### `search_asset` — Step 2 preview, Steps 3a/3b/3c registration

Per the public MCP server reference, parameters are:

- **At least one of:** `searchQuery` (natural-language search) or `maxResults` (integer limit).
- Optional: `assetFilters` (asset types to include — `["llm"]`, `["mcp"]`, `["agent"]`), `statuses`, `organizationId`, `sharedWithMe`, `sortCriteria`, `ascending`, `exchangeScope` (Private / Public).

**Always include `assetFilters` to narrow results.** Searching without it returns noise.

Exchange Viewer permission required.

Example (semantic — match the actual host's parameter shape):
```
search_asset(
  searchQuery: "Slack messaging",
  assetFilters: ["mcp"],
  exchangeScope: "private"
)
```

If the search returns NO results, **do not** invent or add the asset to dependencies. Tell the user: "I didn't find a matching MCP server in Exchange. Want to provide a custom configuration, or skip this asset?"

### `validate_project` — Step 8 (final validation)

Pass the project folder path (`projectPath`). Returns a list of validation findings: schema errors, missing references, malformed expressions.

Two kinds of fixes when errors come back:

1. **Auto-fix safely** — formatting/indentation issues, missing required keys (e.g., `info.version` missing). Apply the fix and re-run.
2. **Ask the user** — schema-required references missing. Example: an MCP in `registry.mcps` is referenced from a node's action, but no matching `context.connections` entry exists. Ask: "Want me to fix the reference in `<file>` line `<n>`, or remove the action from the node?" **Never auto-insert missing assets.**

Loop until clean or user gives up.

### `publish_agent_network_assets` — Step 9

Parameters per the public reference:
- **Required:** `assetVersion` (semver string).
- Optional: `projectPath` (defaults to current), `groupId` (defaults to user's active org).

Prereqs the user needs already:
- ACB authenticated to Anypoint Platform.
- Valid `exchange.json` (no empty `default: ""` for `secret: true` variables — fill them or the publish fails).

After publish, the skill should surface the published asset URLs back to the user.

### `deploy_agent_network` — Step 10

Parameters per the public reference (none strictly required, but the tool prompts for missing values; gather up front for a clean call):
- `projectPath`
- `environmentName` (Anypoint environment — Sandbox, Production, etc.)
- `privateSpaceName` (CloudHub 2.0 private space)
- `ingressGatewayName` (Flex Gateway for inbound traffic)
- `egressGatewayName` (Flex Gateway for outbound traffic)

Prereqs:
- A Managed Omni Gateway in the user's business group.
- Successful publish (or assets already in Exchange).
- Filled-in secrets in `exchange.json`.

After deploy, surface the deployment URL/ID and tell the user where to find logs/metrics.

---

## Degrade gracefully when MCP isn't available

The skill is designed to work in Cursor, standalone Claude Code, Codex — anywhere without the MuleSoft MCP server. Per-step fallbacks:

### Step 2 / Step 3 — asset search

Without `search_asset`:
- Ask the user: *"Do you have this asset in Anypoint Exchange already? If yes, what's its `groupId`/`assetId`/`version`? If no, I'll register a placeholder."*
- For Exchange-mode assets, get the user to provide `groupId`, `assetId`, `version`, and (for MCPs) the transport kind and tool names.
- For unknown assets, register an inline placeholder per Case C in `asset-selection.md`.

### Step 8 — validation

Without `validate_project`:
- Run the skill's structural checklist (every node reachable, every path terminates at echo, every router has `otherwise:`, every `${var}` has a matching `exchange.json` entry, every action referenced in nodes is defined in `actions:`, A2A actions in `reasoning.actions` are bare references, etc.).
- Tell the user: *"I've validated the structure. To run MuleSoft's full schema validator, install the MuleSoft MCP server in your IDE or use the Anypoint CLI agentnetwork plugin (see <https://docs.mulesoft.com/anypoint-code-builder/af-build-agent-networks-in-a-ci-cd-environment>)."*

### Step 9 — publish

Without `publish_agent_network_assets`:
- Tell the user: *"To publish, run the Anypoint CLI from your terminal. See <https://docs.mulesoft.com/anypoint-code-builder/af-publish-agent-network-assets> for current commands."*
- Don't paste a `anypoint-cli` command line — flag spelling drifts across CLI versions; the doc link is more reliable.

### Step 10 — deploy

Without `deploy_agent_network`:
- Tell the user: *"To deploy, use Anypoint Code Builder or run the Anypoint CLI. See <https://docs.mulesoft.com/anypoint-code-builder/af-deploy-agent-network-targets> for current commands."*
- Same caveat — link, don't paste.

---

## What this skill must NOT do

- **Do not call `configure_agent_network_yaml`.** That tool returns instructions for an LLM to follow. We *are* the LLM and we already have our instructions; calling it duplicates or contradicts our own logic.
- **Do not call `create_agent_network_project`.** It would create files we'd then have to overwrite.
- **Do not silently shell out to `anypoint-cli` for auth-bearing actions.** If the MCP tools aren't available, surface the doc link as a printable instruction. Auth-bearing actions need the user's explicit go-ahead.
- **Do not auto-deploy.** Step 8 ends at "validated". Step 9 is publish, only when the user opts in. Step 10 is deploy, only when the user opts in.
- **Do not auto-insert missing assets when validation reports a dangling reference.** Ask the user to fix or remove.
- **Do not fabricate MCP `tool_name` values, asset IDs, or model IDs** even when the MCP server isn't present. Ask the user.
