# Agent Broker Skills

Reusable skills for building, troubleshooting, and migrating MuleSoft Agent Broker (Agent Network V2) projects. They're designed to work in any coding agent: Claude Code, Cursor, Vibes, and others.

## Skills

### [`build-agent-broker-project/`](./build-agent-broker-project/SKILL.md)

An end-to-end guided experience for building a V2 Agent Broker project. It covers:

- functional requirements
- asset registration
- broker definition
- asset assignment
- instruction refinement
- final topology review
- validate, publish, deploy, undeploy, and unpublish
- edit and troubleshoot workflows

The skill uses the Anypoint CLI Agent Fabric plugin first (reflects v1.4.0) and falls back to the MuleSoft MCP tools, then to a no-tool path.

- `SKILL.md`: main entry, phases, lifecycle steps, and edit/troubleshoot workflows (Workflow B)
- `references/canonical-example.md`: the IT Help Investigation example, verbatim from the docs
- `references/gotchas.md`: GA syntax and compile-error rules, asset modes, CLI/MCP tooling, publish versioning, deploy targets and gateways, broker resources, conversational memory
- `references/troubleshooting.md`: runtime symptoms, logging, the CLI error-code catalog, and dependent-service failures

### [`translate-agent-broker-old-to-new-project/`](./translate-agent-broker-old-to-new-project/SKILL.md)

Converts MuleSoft Agent Network V1 projects (`schemaVersion 1.0.0`) to V2 (`agentNetwork 2.0.0`). Each V1 broker becomes a V2 broker with one orchestrator node. Output goes to a sibling folder with a `_v2_conversion` suffix.

- `SKILL.md`: the 4-step workflow
- `references/v1-example.md`, `references/v2-example.md`, `references/v2-template.agent`: reference material

## Source of truth

The authoritative source is the Agent Network 2.0 documentation: [`mulesoft/docs-code-builder`](https://github.com/mulesoft/docs-code-builder/tree/latest/agent-network/2.0), branch `latest`. The canonical sample is the docs page *Example: Building an IT Investigation Broker*. A runnable copy is in [`MuleSoft-AI-Chain-Project/example-mule-apps`](https://github.com/MuleSoft-AI-Chain-Project/example-mule-apps/tree/master/agent-network-2.0-examples/it-investigation-broker-example) at `agent-network-2.0-examples/it-investigation-broker-example`.
