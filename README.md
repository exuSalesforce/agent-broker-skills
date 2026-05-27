# Agent Broker Skills

Reusable skills for building and migrating MuleSoft Agent Broker (Agent Network V2) projects. Designed to work in any coding agent — Claude Code, Cursor, Vibes, etc.

## Skills

### [`mulesoft-agent-broker-builder/`](./mulesoft-agent-broker-builder/SKILL.md)

End-to-end guided experience for building a V2 Agent Broker project: functional requirements → asset registration → broker definition → asset assignment → instruction refinement → final topology review, plus publish and deploy. Replaces the `create_agent_network_project` and `configure_agent_network_yaml` MCP tools while keeping integration with `validate_project`, `search_asset`, `publish_agent_network_assets`, and `deploy_agent_network`.

- `SKILL.md` — main entry, 6 phases + publish/deploy + edit modes
- `references/agent-script-grammar.md` — gotchas, compile-error rules
- `references/canonical-example.md` — verbatim IT Help Network example
- `references/asset-selection.md` — RULE-ASSET-MODE, decision trees
- `references/mcp-integration.md` — behavior when MuleSoft MCP tools are present

### [`mulesoft-agent-broker-v1-to-v2-converter/`](./mulesoft-agent-broker-v1-to-v2-converter/SKILL.md)

Converts MuleSoft Agent Network V1 projects (`schemaVersion 1.0.0`) to V2 (`agentNetwork 2.0.0`). Each V1 broker becomes a V2 broker with one orchestrator node. Output is written to a sibling folder with `_v2_conversion` suffix.

- `SKILL.md` — 4-step workflow
- `references/v1-example.md`, `v2-example.md`, `v2-schema.md` — reference material

## Source of truth

The canonical V2 sample is the **IT Help Network** at <https://github.com/mulesoft-emu/agent-fabric-specification/tree/e-xu_sfemu-patch-1/examples/agent-script/it-help-network>. When the Beta Guide, MCP server templates, and the canonical example disagree, the canonical example wins.
