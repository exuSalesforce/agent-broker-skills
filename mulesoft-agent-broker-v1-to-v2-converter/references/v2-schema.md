![][image1]

Agent Broker (Agent Script) Beta Guide

| MuleSoft, Spring ’26 |
| :---- |

Last updated: April 30, 2026

© Copyright 2000–2026 Salesforce, Inc. All rights reserved. Salesforce is a registered trademark of Salesforce, Inc., as are other names and marks. Other marks appearing herein may be trademarks of their respective owners.

# CONTENTS

[Introduction to Agent Networks (2.0)	3](#introduction-to-agent-networks-\(2.0\))

[Get Started with Agent Networks (2.0)	10](#get-started-with-agent-networks-\(2.0\))

[Create an Agent Network (2.0)	10](#create-an-agent-network-\(2.0\))

[Define an Agent Network Specification (2.0)	11](#define-an-agent-network-specification-\(2.0\))

[Publish an Agent Network Project (2.0)	13](#publish-an-agent-network-project-\(2.0\))

[Deploy an Agent Network (2.0)	13](#deploy-an-agent-network-\(2.0\))

[Build Agent Networks in a CI/CD Environment	13](#build-agent-networks-in-a-ci/cd-environment)

[Agent Network 2.0 Project File Reference	14](#agent-network-2.0-project-file-reference)

[Appendix: Building an IT Investigation Broker	67](#appendix:-building-an-it-investigation-broker)

| ![][image2] | Important: Agent Network 2.0  is a pilot or beta service that is subject to the Beta Services Terms at [Agreements \- Salesforce.com](https://www.salesforce.com/company/legal/) or a written Unified Pilot Agreement if executed by Customer, and applicable terms in the [Product Terms Directory](https://ptd.salesforce.com/). Use of this pilot or beta service is at the Customer's sole discretion.  |
| :---- | :---- |

# Introduction to Agent Networks (2.0)  {#introduction-to-agent-networks-(2.0)}

An agent network coordinates groups of agents, brokers, LLMs, and MCP servers and acts as a central hub for defining, validating, and executing agentic processes across your enterprise.

An agent network provides the building blocks of your agentic deployment. At the start of a new agent network project, we provide a YAML template and Agent Script file to give you a head start. You can even use MuleSoft Vibes to configure your network, publish the assets to Anypoint Exchange, and deploy your agent network instance. This approach abstracts away the underlying technical complexities, allowing you to focus on the business constraints and context of your process without needing to understand the inner workings of the orchestration engine.

## Key Benefits of Agent Networks

Create robust, automated agent networks with these key benefits.

**Simplified Development**  
Define your agent processes in an intentional way using a simple YAML file, which is easier and faster than writing complex code.

**Safety and Governance**  
Ensure control and compliance across every agent and tool interaction with enterprise-grade governance.

**Observability**  
Gain insight into agent actions with a visual trace of their decision-making.

**Flexibility**  
Coordinate any type of agent and tool, regardless of where it’s built and deployed so you can build a diverse network of agents.

**Tool and Agent Invocation**  
Manage the invocation of both deterministic (tool-based) and probabilistic (LLM-based) actions.

## The Hybrid Determinism Pattern

Brokers in Agent Network 2.0 architecture leverages a hybrid deterministic approach by separating steps requiring probabilistic LLM reasoning from those requiring deterministic control-flow logic.

* Probabilistic (LLM-based) nodes handle complex, open-ended decisions that require nuanced judgment, such as classification, or AI powered thinking.  
* Deterministic nodes: manage all remaining predefined execution paths, ensuring a fixed and guaranteed workflow once a key probabilistic judgment has been finalized.

This structure allows brokers to apply high-level LLM intelligence where complex judgment is needed, while the overall agent graph maintains reliable, rule-based predictability across the operational flow.

## Agent Network Components

An agent network is a collection of brokers, agents, LLMs, and MCP servers that are connected to each other. 

**Agent Brokers**  
An intelligent routing service that coordinates task delegation across A2A-compliant agents in your enterprise. You define a broker and its nodes in Agent Script. Nodes are connected in a graph that encapsulates all steps that describe the orchestration of agents, tools, and large language models. A graph-based approach to agent brokers ensures that connected paths guarantee a specific order of operations and enable more complex orchestration that combines deterministic and non-deterministic elements. 

* **Nodes**  
  Each “step” within a graph. Nodes can be used to orchestrate actions with LLM powered reasoning, or perform deterministic action like routing. 

* **Trigger**  
  An entry point into a graph. Triggers specifies events, calls, or messages that execute workflows. All graphs must start on an A2A trigger. 

* **Edges**  
  Transitions from node to node. Input data enters the node and output data exits the node.

* **Agents**  
  An autonomous software component that uses goals, context, and available tools, via a large language model (LLM), to decide and execute actions on behalf of a user or system. Your agent network can use both locally defined and externally defined agents to complete tasks.

  Agents must be compliant with the A2A specification. 

* **MCP servers**  
  A service that implements the Model Context Protocol (MCP) to expose tools and data to AI clients, enabling LLMs to invoke external capabilities through a standard interface. MCP servers can be defined either locally in the agent network or externally in a different agent network or elsewhere in your company. Your agent network can use both locally defined and externally defined MCP servers to complete tasks.

**Registry**  
A section within an agent network project that defines the assets, connections, and policies necessary to build, execute, and govern a Broker. Assets defined in this section of an agent network file are published to Exchange and become reusable. You can also reference existing assets from Exchange in a graph without needing to define them in the registry section.

## Agent Network YAML and Agent Script {#agent-network-yaml-and-agent-script}

An agent network project defines a structured configuration for multi-agent systems, enabling orchestration of AI agents with external services, tools, and inter-agent communication. This format provides a declarative way to define agent capabilities, dependencies, and service integrations.

An agent network projects include these files.

| `/my-agent-network-project | - exchange.json | - agent-network.yaml | /brokers || - broker1.agent` |
| :---- |

* `exchange.json`: Contains asset metadata available in Anypoint Exchange after publishing your agent network assets.  
* `agent-network.yaml`: Contains a registry section that defines Exchange assets used in your project and a context section that defines connections and policies for the project.  
* `.agent` files: Contains broker and node definitions and configurations that enable multi-agent orchestration in your project.

To learn more about agent-network.yaml and Agent Script references, see the [Agent Network Project File Reference (2.0)](#agent-network-2.0-project-file-reference) in this guide.

For a full example that includes registry and node definitions, see [Appendix: Building an IT Investigation Broker](#appendix:-building-an-it-investigation-broker).

## Agent Network Architecture

This diagram shows an agent network with agents and MCP servers, traffic governed by Flex Gateway, and observed in Anypoint Monitoring and Agent Visualizer.

![][image3]

\[alt text: Agent network showing agents and MCP servers defined in YAML and published to Exchange.\]

1. Publish the agentic assets to Anypoint Exchange for discovery and reuse after you define the agent network (agents and MCP servers) in the agent graph YAML in Anypoint Code Builder.  
2. Deploy the agentic assets to CloudHub 2.0 or Runtime Fabric (managed in Runtime Manager).  
3. Enforce policies on incoming traffic to the network with an ingress Flex Gateway, which sits in front of broker and API endpoints.  
4. Enforce policies, manage connections, and emit telemetry data with an egress Flex Gateway, which sits on outbound paths from agents to external agents and services.  
5. Collect logs, metrics, and traces from Flex Gateway and runtimes in Anypoint Monitoring.

## Agent Network Assets in Anypoint Exchange

The following agent network asset types are supported in Exchange.

* **Agent Network**  
  An agent network project is published in Exchange as an Agent Network asset. The asset is a .zip file that contains all project files. To learn more, see [Agent Network YAML and Agent Script](#agent-network-yaml-and-agent-script) or the Agent Network Project File Reference (2.0) in this guide.

* **Agents**  
  Programs that perform tasks autonomously or semiautonomously. AI agents use the Agent2Agent (A2A) protocol to communicate and collaborate with each other to perform tasks.

  When you publish agent network projects to Exchange, the individual agentic assets defined in the agent network project file are registered as either agents or MCP servers. Brokers are published with the agent asset type and are automatically tagged as brokers for easy identification.

* **LLMs (Large Language Models)**  
  AI assets for processing, understanding, and generating human readable language. Agent network projects [support the various LLMs](#large-language-models). The LLM asset type defines only the provider and contract information. You define the model and connectivity details in the agent network file.  
* **MCP Servers**  
  Applications or APIs exposed through the Model Context Protocol (MCP). MCP is an open protocol designed to standardize how applications provide context and capabilities to LLMs and AI agents. In the agent network file, MCP servers assets are defined as `Tools`.

## Large Language Models {#large-language-models}

Agent networks support the latest Gemini and OpenAI models.

* Azure OpenAI and OpenAI API:  
  * GPT-5.2  
  * GPT-5.2 Pro  
  * GPT-5-mini  
  * GPT-5 nano  
  * GPT-5.2 Pro  
  * GPT-5  
* Gemini API:  
  * Gemini 2.5 Pro  
  * Gemini 2.5 Flash-Lite  
  * Gemini 2.5 Flash  
  * Gemini 3 Flash  
  * Gemini 3 Pro

For more information about these models, see the [OpenAI](https://developers.openai.com/api/docs/models) and [Gemini](https://ai.google.dev/gemini-api/docs/models) documentation.

The following table details the requirements and recommended models.

| Model Provider | Required Endpoint | Required Capabilities | Suggested Models |
| :---- | :---- | :---- | :---- |
| OpenAI | /responses | Reasoning Native structured output Function and Custom tool calling | For lower latency: GPT-5-mini For complex reasoning: Evaluate models |
| Gemini | /generateContent (Native API) | Native Thinking (via thinkingBudget and thinkingLevel) Native structured output (responseSchema) Function and custom tool calling | For lower latency: Gemini 2.5 Flash, Gemini 2.5 Flash-Lite For complex reasoning: Gemini 3 Pro (Deep Think capabilities) |

Agent network supports text-based prompts and responses. Image and binary message types aren’t supported.

## A2A Protocol

The Agent2Agent (A2A) Protocol governs agent-to-agent communication. This protocol powers orchestration, observability, and governance features in agent networks. MuleSoft supports v0.3.0 of the A2A Protocol Specification.

### **Context and Task ID Scoping in Agent Networks**

In MuleSoft agent networks, the brokers receiving a request always generate a `contextId` and `taskId`. These IDs define the state and scope of a specific conversation between two agents. A `taskId` is always matched to a `contextId`, but a `contextId` can exist without a `taskId`.

In a multi-agent network, a client sends a request to Broker\_1 and Broker\_1 generates the necessary IDs for that request. When Broker\_1 sends a new request to the next broker or non-broker agent in line, that broker or non-broker agent establishes a unique `contextId` and `taskId` for the new request.

Consider a network with a client and two broker nodes (1 and 2).

* The IDs used between the client and Broker\_1 are independent of the IDs used between Broker\_1 and Broker\_2.  
* When Broker\_1 delegates a task to Broker\_2, Broker\_2 (acting as a server) generates its own `contextId` and `taskId`.  
* Broker\_1 is responsible for maintaining a mapping between its own upstream `taskId` (used to respond to its client) and the downstream taskId it’s tracking with Broker\_2.  
* If Broker\_2 requires more information (if it returns status: `input-required`), it provides a `contextId` and `taskId` to Broker\_1. Broker\_1 uses these IDs to provide the requested input to Broker\_2. The client never sees Broker\_2’s internal IDs.

| Relationship | Role | Logic |
| :---- | :---- | :---- |
| Client → Broker\_1 | Broker\_1 is server | Generates `contextId_1` and `taskId_1` for the client. |
| Agent A → Broker\_2 | Broker\_2 is server | Generates `contextId_2` and `taskId_2` for Broker\_1. |
| Network Broker | Broker\_1 | Broker\_1 maps `contextId_1` and `taskId`\_1 to `contextId_2` and `taskId_2`. |

For more information, see [Life of a Task \- Group Related Interactions](https://a2a-protocol.org/v0.3.0/topics/life-of-a-task/#group-related-interactions).

# Get Started with Agent Networks (2.0) {#get-started-with-agent-networks-(2.0)}

Creating agent networks is a multi-step process. For this beta, review the prerequisites and task overviews to make sure you can create agent networks efficiently. For more information, see [Get Started with Agent Networks](https://docs.mulesoft.com/anypoint-code-builder/af-get-started). 

# Create an Agent Network (2.0) {#create-an-agent-network-(2.0)}

MuleSoft Vibes or Anypoint Code Builder includes all the agent network functionality you need to get started. 

## Before You Begin

Make sure you review the [prerequisites](https://docs.mulesoft.com/anypoint-code-builder/af-get-started#before-you-begin).

## Create a Network Using MuleSoft Vibes

MuleSoft Vibes can help you create your project. 

1. In the Anypoint Code Builder activity bar, click the agent icon.  
2. Describe your agent network, including the brokers, agents, MCP servers, and LLMs you want to connect. MuleSoft Vibes does the rest.

To get started try one of these suggested prompts.

* Create a new agent network project called "Employee Onboarding." Add HR agent and CRM agent.  
* Build a travel planner broker that plans multi-city trips and generate a new agent network.  
* Generate a new agent network project called IT Help Investigation for incident management.

## Create a Network Using Anypoint Code Builder

1. If you're not logged in, log in to your Anypoint Platform account.  
2. In Anypoint Code Builder choose one of the following:  
   - In Create, select *Create an Agent Network*.  
   - From the Command Palette, run this command: *MuleSoft: Create an Agent Network Project*.  
3. Enter a name for the project and select the location to create the project.  
4. Select the business group associated with the target space you created in [Get Started with Agent Networks](http://af-get-started.md). The business group you select must be the same business group you selected when you created the target space.  
5. Select *Create Project*.

MuleSoft Vibes and Anypoint Code Builder creates these files as part of your agent network project. These files define your agent network specification to meet your business requirements. 

| `/my-agent-network-project | - exchange.json | - agent-network.yaml | /brokers || - broker1.agent` |
| :---- |

* `exchange.json`: Contains asset metadata available in Anypoint Exchange after publishing your agent network assets.  
* `agent-network.yaml`: Contains a registry section that defines Exchange assets used in your project and a context section that defines connections and policies for the project.  
* `.agent` files: Contain broker and node definitions and configurations that enable multi-agent orchestration in your project.

For more information, see [Agent Network Project File Reference](#agent-network-2.0-project-file-reference).

# Define an Agent Network Specification (2.0) {#define-an-agent-network-specification-(2.0)}

After you create your agent network project, configure `agent-network.yaml`,  `exchange.json`, and your Agent Script files to reflect the structure of your project.

## Define a Network Using MuleSoft Vibes

MuleSoft Vibes can help you configure your agent network specification. For more information, see [API AI Create Spec](anypoint-code-builder::api-ai-create-spec.adoc).

1. In the Anypoint Code Builder activity bar, click the agent icon.  
2. Give the agent information about your agent network, including the brokers, agents, MCP servers, and LLMs you want to connect.

To get started try one of these suggested prompts.

* In my new agent network project called IT Help Investigation, I need a broker that triages incoming support tickets. It should classify ticket severity (High, Low) using an LLM. If the ticket is too vague, the agent should ask the user for clarification before classifying. High-severity tickets get escalated via an MCP tool. Low-severity tickets require multi-agent investigation using a Help Center agent and a License Procurement agent. The severity classification should be LLM-based, but routing after classification must be deterministic.  
* Please update the instructions for the Cross-Platform Triage agent to be more concise.  
* Add another condition in the severity router for medium.

## Define a Network Using Anypoint Code Builder or IDE

If you don't want to use MuleSoft Vibes, use Anypoint Code Builder or your IDE to edit the `agent-network.yaml`,  `exchange.json`, and your Agent Script files and define your agent network and authentication.

To understand sections of the project files and expected values, see [Agent Network Project File Reference](#agent-network-2.0-project-file-reference). 

If your agent network references Anypoint Exchange assets, you need to use the asset IDs in `exchange.json` to add references in the `agent-network.yaml` file. For more information, see [Add Exchange Assets to Your Agent Network Project](#add-exchange-assets-to-your-agent-network-project).

## Add Exchange Assets to Your Agent Network Project {#add-exchange-assets-to-your-agent-network-project}

If you have existing Exchange assets you want to use in your agent network, add them to the dependencies attribute in `exchange.json` in your project. After you add assets, edit the `agent-network.yaml` and Agent Script files to indicate which brokers use those assets.

### **Add Assets Using MuleSoft Vibes**

1. In the Anypoint Code Builder activity bar, click the agent icon.  
2. Tell the agent that you want to add Exchange assets to your project. MuleSoft Vibes does the rest.

To get started try one of these suggested prompts.

* Add tools for ticket tracking in my IT Help Investigation project.  
* Find the Rootly MCP server from Public Exchange and add it to my IT Help Investigation project. Apply a rate limiting policy.

### **Add Assets Using Anypoint Code Builder**

1. If you're not logged in already, log in to your Anypoint Platform account.

2. In your Anypoint Code Builder project, choose one of the following:

* In Explorer, right-click a project file and select **Add Exchange Assets to Agent Network Project**.  
* In the Command Palette, run this command: **MuleSoft: Add Agent Network Assets to Agent Network Project**.

3. In Add Exchange Assets to Project, enter information about the assets to add.

4. Select **Add to Project**.

   Assets are added to the `dependencies` attribute in the `exchange.json` file in your project.

5. Edit the `agent-network.yml` and Agent Script files to indicate which brokers use those assets.

If you added a dependency that doesn't belong to the same business group as your agent network, then in `agent-network.yaml`, specify the `namespace` property at the same level as name. Provide the `groupId` (business group ID) for the business group that the dependency belongs to. The `groupId` value is the same as the `groupId` value of the corresponding dependency in `exchange.json`.

If you don't provide a namespace value, the same `groupId` as the agent network project is used.

After you add dependencies, they're available as values in auto-completion menus in Anypoint Code Builder. For example, after you add `test-agent` to `exchange.json`, the value `test-agent` is available in auto-completion menus in the code editor in `agent-network.yaml` when you reference an agent.

# Publish an Agent Network Project (2.0) {#publish-an-agent-network-project-(2.0)}

Build and publish your agent network project as Anypoint Exchange assets. When you publish, an asset is created for each broker, agent, and MCP server that’s in your agent network. For more information, see [Publish Agent Network Assets](https://docs.mulesoft.com/anypoint-code-builder/af-publish-agent-network-assets).

# Deploy an Agent Network (2.0) {#deploy-an-agent-network-(2.0)}

Deploy your agent network instance to a deployment target. You can deploy to a CloudHub 2.0 private space or to a Runtime Fabric. When you deploy, Flex Gateway secures your agent network into the ingress gateway. Also, it secures your brokers, agents, and MCP servers into the egress gateway. For more information, see [Deploy Agent Network Instances](https://docs.mulesoft.com/anypoint-code-builder/af-deploy-agent-network-targets).  

# Build Agent Networks in a CI/CD Environment {#build-agent-networks-in-a-ci/cd-environment}

If you run operations within a CI/CD environment, you can use Anypoint CLI's plugin to set up, create, build, publish, and deploy agent networks. For more information, see [Build Agent Networks in a CI/CD Environment](https://docs.mulesoft.com/anypoint-code-builder/af-build-agent-networks-in-a-ci-cd-environment).

# 

# Agent Network 2.0 Project File Reference {#agent-network-2.0-project-file-reference}

An agent network project defines a structured configuration for multi-agent systems, enabling orchestration of AI agents with external services, tools, and inter-agent communication. This format provides a declarative way to define agent capabilities, dependencies, and service integrations.

An agent network projects include these files.

| `/my-agent-network-project | - exchange.json | - agent-network.yaml | /brokers || - broker1.agent` |
| :---- |

* `exchange.json`: Contains asset metadata available in Anypoint Exchange after publishing your agent network assets.  
* `agent-network.yaml`: Contains a registry section that defines Exchange assets used in your project and a context section that defines connections and policies for the project.  
* `.agent` files: Contains broker and node definitions and configurations that enable multi-agent orchestration in your project.

* [Agent Network (2.0) YAML Reference](#agent-network-\(2.0\)-yaml-reference)  
* [Agent Script Reference](#agent-script-reference)

For a full example that includes registry and node definitions, see [Appendix: Building an IT Investigation Broker](#appendix:-building-an-it-investigation-broker).

## Agent Network (2.0) YAML Reference  {#agent-network-(2.0)-yaml-reference}

The agent-network.yaml details the required structure and properties that define your project's assets, connections, and policies.

[Agent Network Section](#agent-network-section)

[Info Section](#info-section)

[Registry Section](#registry-section)

[Context Section](#context-section)

[Brokers Section](#brokers-section)

[Expressions Format](#expressions-format)

[Exchange JSON File](#exchange-json-file)

### **Agent Network Section** {#agent-network-section}

agent-network

This is the root section of the agent network. Must include at least one of `registry`, `context`, or `brokers` must be present.

This section has these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| info | Provides metadata about the graphs.  | [info Object](#info-section) | No |
| agentNetwork | This string must be the [version number](#versions) of the agent network specification used by the agent network document. This is not related to the document's `info.version` property.  | String | Yes |
| registry | Definitions of agents, agents, and MCPs, LLMs. | [registry](#registry-section) Object | No |
| context | Reusable entities scoped to the graphs defined in this file (for example, connections and policies). | Object | No |
| brokers | A mapping of runtime Agent Script definitions that can be executed by the platform. | [brokers Object](#brokers-section) | No |

### **Info Section** {#info-section}

info

This section contains basic metadata about the agent network document itself.

**Example**

| `info:   label: Employee Onboarding Workflow   description: |     A multi-agent network for employee onboarding. Orchestrates HR system setup,     Salesforce profile creation, laptop and badge requests, and Slack notifications     for new hires.   version: 1.0.0   tags:     - employee-onboarding     - multi-agent     - hr     - production` |
| :---- |

The section has these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| label | The human-readable name of the agent network. | String | Yes |
| version | The version number of the `agent-network.yaml` file. | String | Yes |
| description | A human readable summary of what the agent network does.  Accepts CommonMark syntax. | String | No |
| tags | Categorization tags for this agent network document. | Array of Strings | No |
| `summary` | A short summary of the purpose of the agent network. | String | No |
| `termsOfService` | A URI for the Terms of Service for the API. This must be in the form of a URI. | String | No |
| `contact` | The contact information for the agent network. | Object | No |
| `contact.name` | The identifying name of the contact person/organization. | String | No |
| `contact.url` | The URI for the contact information. This must be in the form of a URI. | String | No |
| `contact.email` | The email address of the contact person/organization. This must be in the form of an email address. | String | No |
| `license` | The license information for the agent network. | Object | No |
| `license.name` | The license name used for the agent network. | String | Yes (if license present) |
| `license.identifier` | An SPDX license expression. Mutually exclusive with `license.url`. | String | No |
| `license.url` | A URI for the license. This must be in the form of a URI. Mutually exclusive with `license.identifier`. | String | No |

### **Registry Section** {#registry-section}

registry

Use this section to organize and reference reusable agents, LLMs, and MCP tools. The registry section defined in the file is considered the “local registry”. Assets defined in the registry are published to Exchange. 

**Example** 

| `registry:   agents:     hr-system-agent:       info:         label: HR System Agent         description: Agent that creates new-hire records and manages HR system setup.       metadata:         platform: AgentForce         protocol: a2a         card:           a2a:             protocolVersion: "0.3.0"             name: hr-system-agent             description: Creates and manages employee records in the HR system.             url: https://hr-agent.example.com/a2a             defaultInputModes: [application/json, text/plain]             defaultOutputModes: [application/json, text/plain]         tools:           - mcp:               ref:                 name: slack-mcp               allowed: [sendMessage, listChannels]         llm:           ref:             name: Open-AI-LLM     salesforce-agent:       info:         label: Salesforce Onboarding Agent         description: Agent that provisions Salesforce profiles for new hires.       metadata:         platform: AgentForce         protocol: a2a         card:           a2a:             protocolVersion: "0.3.0"             name: salesforce-agent             description: Onboards new employees to Salesforce.             url: https://sfdc-agent.example.com/a2a             defaultInputModes: [application/json]             defaultOutputModes: [application/json]         llm:           ref:             name: Open-AI-LLM   mcps:     slack-mcp:       info:         label: Slack MCP Server         description: MCP server for sending messages and listing Slack channels.       metadata:         protocolVersion: "2024-11-05"         transport:           kind: sse           ssePath: /mcp         tools:           - name: sendMessage             description: Send a message to a Slack channel.             inputSchema:               type: object               properties:                 text: { type: string }                 channelId: { type: string }           - name: listChannels             description: List available Slack channels.             inputSchema:               type: object   llms:     Open-AI-LLM:       info:         label: OpenAI LLM         description: OpenAI provider for orchestration and generation nodes.       metadata:         platform: OpenAI         models:           - gpt-4o           - gpt-4o-mini           - gpt5-mini     Azure-OpenAI-LLM:       info:         label: Azure OpenAI LLM         description: Azure OpenAI for orchestration nodes.       metadata:         platform: AzureOpenai         models:           - gpt-4o` |
| :---- |

The `registry` section has these properties.

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `agents` | The list of agents defined as part of this network. | Object | No | Keys: identifiers matching ^\[a-zA-Z\_\]\[a-zA-Z0-9\_.-\]\*$. Values: AgentEntity |
| `mcps` | The list of MCP servers defined as part of this network. | Object | No | Keys: identifiers. Values: MCPServerEntity |
| `llms` | The list of LLM providers defined as part of this network. | Object | No | Keys: identifiers. Values: LLMEntityEach `LLMEntity` requires `metadata.platform`: `Gemini`, `OpenAI`, or `AzureOpenai`. |

#### **Info** 

All nodes in this section share these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `label` | The human readable short text. | String | No |
| `description` | A human readable text of what this element does. Accepts CommonMark syntax. | String | No |
| `tags` | Optional tags. | Array of Strings | No |

#### 

#### **Agents**

Each registry agent is an `AgentEntity`. The schema requires a `metadata` object. The A2A agent card, tools, and LLM reference are nested under `metadata`, not at the root of the agent.

The agents section has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `agents.info` | Metadata for the agent. | InfoObject | No |
| `agents.metadata` | Platform, protocol, card, tools, and LLM wiring for the agent.  | Object | Yes |
| `agents.metadata.platform` | Host platform for the agent (for example, AgentForce or Bedrock). | String | Yes |
| `agents.metadata.protocol` | Integration protocol for the agent; use values such as a2a or other as defined by the platform. | String  | Yes |
| `agents.metadata.card`  | Agent card payload. Include a2a for an A2A card, or other for other card types.  | Object | No |
| `agents.metadata.card.a2a`   | The A2A agent card. See [A2A Card](https://docs.google.com/document/d/1lKW5MfLAkK5N1IlgUS7MNSSn0k6jYZEP1ZO4WGvaiOU/edit#a2a-card). | AgentCard | Yes (when using the a2a card branch) |
| `agents.urls`  | Named URLs for the agent. | Array | No |
| `agents.metadata.tools`  | The list of tool providers (MCP or A2A) | Array\[Object\] | No |
| `agents.metadata.llm`  | Reference to the LLM used by this agent (ref to a declared LLM) | Object | No |

Authentication for outbound calls is configured on [`connections`](#connections) (and related policies), not on registry `AgentEntity` definitions.

##### A2A Card {#a2a-card}

This section adheres to the Agent-to-Agent (A2A) specification v0.3.0 and describes an agent’s contract, skills, and capabilities. This is a standard A2A agent card [as defined in the Agent2Agent (A2A) Protocol specification](https://a2a-protocol.org/latest/specification/#55-agentcard-object-structure).

The A2A card section has these properties. 

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `name` | Human-readable name for the agent. | string | Yes | — |
| `description` | Human-readable description of the agent's purpose. | string | Yes | — |
| `url` | Preferred endpoint URL for interacting with the agent. Must support preferredTransport. | string | Yes | e.g. https://api.example.com/a2a/v1 |
| `version` | Agent's own version number (format defined by provider). | String | Yes | e.g. 1.0.0 |
| `protocolVersion` | Version of the A2A protocol this agent supports. | String | Yes | Default: 0.3.0 |
| `capabilities` | Optional capabilities supported by the agent. | AgentCapabilities | Yes | See below |
| `defaultInputModes` | Default supported input MIME types for all skills (overridable per skill). | Array of string | Yes | — |
| `defaultOutputModes` | Default supported output MIME types for all skills (overridable per skill). | Array of string | Yes | — |
| `skills` | The set of skills or distinct capabilities the agent can perform. | Array of AgentSkills | Yes | See below |
| `preferredTransport` | Transport for the main url. Must be available at that URL. | String | No | JSONRPC, GRPC, HTTP+JSON (default: JSONRPC) |
| `additionalInterfaces` | Additional transport+URL combinations for the same agent. | Array of AgentInterface | No | Each: { transport: string, url: string } |
| `additionalInterfaces.transport`  | The transport protocol for this interface. | String | Yes | `JSONRPC, GRPC, HTTP+JSON)` |
| `additionalInterfaces.url` | The URL for this additional interface. | String (URI) | Yes | Valid URI string |
| `provider` | Agent's service provider. | AgentProvider | No | { organization: string, url: string } |
| `documentationUrl` | Optional URL to the agent's documentation. | String | No | — |
| `iconUrl` | Optional URL to an icon for the agent. | String | No | — |
| `security` | Security requirement objects for all interactions (OpenAPI 3.0 style; OR of ANDs). | Array of object | No | Each object: scheme names → array of scope strings |
| `securitySchemes` | Declared security schemes (key \= scheme name). OpenAPI 3.0 Security Scheme Object. | Object | No | apiKey, http, oauth2, openIdConnect, mutualTLS |
| `signatures` | JSON Web Signatures for this AgentCard (RFC 7515 JWS). | Array of AgentCardSignature | No | { protected, signature, header? } |
| `supportsAuthenticatedExtendedCard` | If true, the agent can return an extended card to authenticated users. | Boolean | No | Default: false |

###### *Skills Properties*

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| `id` | Unique identifier for the skill. | String | Any string value | No |
| `name` | A human-readable name for the skill. | String | Any string value | No |
| `description` | A description of what this skill does. | String | Any string value | No |
| `examples` | Usage examples demonstrating how to use this skill. | Array of strings | Array of example strings | No |
| `inputModes` | Supported input MIME types for this skill (overrides defaultInputModes). | Array of strings | Array of MIME type strings | No |
| `outputModes` | Supported output MIME types for this skill (overrides defaultOutputModes). | Array of strings | Array of MIME type strings | No |
| `tags` | Categorization tags for this skill. | Array of strings | Array of tag strings | No |

###### *Capabilities Properties*

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| `streaming` | Indicates if the agent supports streaming responses. | Boolean | `true` or `false` | No |
| `pushNotifications` | Indicates if the agent supports push notifications. | Boolean | `true` or `false` | No |
| `stateTransitionHistory` | Indicates if the agent maintains state transition history. | Boolean | `true` or `false` | No |
| `extensions` | List of extension capabilities supported by the agent. | Array | Array of extension objects  | No |

#### **MCP**

The MCP Server section has these properties.

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `info` | Metadata for the MCP Server | `InfoObject` | No | – |
| `urls` | Named URLs for the agent. array No  | Array  | No  | –  |
| `metadata` | MCP server descriptor (how to connect and what it exposes). | Object | Yes | – |
| `metadata.protocolVersion` | Version of the MCP protocol. | String | No | "2024-11-05", "2025-03-26", "2025-06-18", "2025-11-25" |
| `metadata.transport` | Transport used for communication. | MCPTransport | Yes | SseTransport, StreamableHttpTransport, or StdioTransport |
| `metadata.provider` | Service provider of the MCP server. | Object | No | { organization: string, url: string } |
| `metadata.capabilities` | Server capabilities. | ServerCapabilities | No | completions, experimental, tasks, logging, prompts, resources, tools |
| `metadata.tools` | List of tools. | Array | No | action definitions |
| `metadata.resources` | List of resources. | Array | No | Resource definitions |
| `metadata.resourceTemplates` | List of resource templates. | Array | No | ResourceTemplate definitions |
| `metadata.prompts` | List of prompts. | Array | No | Prompt definitions |
| `metadata.platform` | Platform the MCP server runs on. | String | No | — |
| `metadata.securitySchemes` | Security schemes for authentication. | Object | No | SecurityScheme by key |

##### MCP Transport types

**Example**

| `registry:   mcps:     weather-mcp:       metadata:         protocolVersion: "2025-06-18"         transport:           kind: streamableHttp           path: /weather/mcp         provider:           organization: Acme Inc.           url: https://www.acme.com     my-mcp-sse:       metadata:         transport:           kind: sse           ssePath: /mcp/sse           messagesPath: /mcp` |
| :---- |

###### 

###### *SseTransport*

Sse Transport has these properties.

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `kind` | Transport type. | String | Yes | `"sse"` |
| `ssePath` | Path to the SSE endpoint. | String | Yes | — |
| `messagesPath` | Path to the messages endpoint. | String | No | Agent Graph Expression Optional |

###### *StreamableHttpTransport*

StreamableHttpTransport has these properties.

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `kind` | Transport type. | String | Yes | "streamableHttp" |
| `path` | Path to the MCP endpoint. | String | No | — |

###### *StdioTransport*

StdioTransport has these properties.

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `kind` | Transport type. | String | Yes | `"stdio"` |
| `instructions` | Instructions to run the MCP server. | String | No | — |

#### **Reference Types** 

**Example**

| `registry:   agents:     myEvaluationAgent:       type: a2a_agent       protocol: a2a       platform: agentforce       kind: evaluation       connections:         - kind: mcp           ref:             assetId: my-mcp-server             version: 1.0.0           allowed:             - tool-name-1       provenance:         kind: exchange         metadata:           organizationId: my-org` |
| :---- |

Reference types used in the Registry section have these properties. 

| Reference | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `AgentRef` | Reference to an agent. | Object | No | `{ name: string, namespace?: string }` |
| `MCPRef` | Reference to an MCP server. | Object | No | `{ name: string, namespace?: string }` |
| `LLMRef` | Reference to an LLM provider. | Object | No | `{ name: string, namespace?: string }` |
| `ConnectionRef` | Reference to a connection. | Object | No | `{ name: string }` |
| `PolicyRef` | Reference to a policy. | Object | No | `{ name: string, namespace?: string }` |

### **Context Section** {#context-section}

context

This section is for reusable entities scoped to the graphs defined in the `agent-network.yaml` file. It holds definitions that are used by the document (for example, connections, policies) but are not published to the agent registry. 

**Example**

| `context:   connections:     hr-system-agent:       kind: a2a       ref:         name: hr-system-agent       url: https://hr-agent.example.com/a2a     salesforce-agent:       kind: a2a       ref:         name: salesforce-agent       url: https://sfdc-agent.example.com/a2a     slack-mcp:       kind: mcp       ref:         name: slack-mcp       url: https://mcp.example.com/slack     Open-AI-LLM:       kind: llm       ref:         name: Open-AI-LLM       url: https://api.openai.com/v1       authentication:         kind: apiKey         apiKey: "${env.OPENAI_API_KEY}"         # optional: headerName (default Authorization)       # optional: policies   policies:     retry-policy:       ref:         name: retry-policy       configuration:         maxAttempts: 3         backoffMs: 1000` |
| :---- |

The `context` section has these properties. 

| Parameter | Description | Type |
| :---- | :---- | :---- |
| connections | The list of connections defined as part of this network. | Connection Objects |
| policies | Policy definitions that govern connection behavior, access, and execution. | Policies Object |

#### **Connections**  {#connections}

connections  
Connection definitions live in the `context` object. Each `connections` object has a kind field that determines its connection type: either `a2a`, `llm`, or mcp. 

Connection objects have these properties.

| Field Name | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `kind` | Connection type | `mcp`, `a2a`, `or llm` | Yes |
| `ref` | A reference to an asset representing the target system. | Reference Object | Yes |
| `url` | The base URL of the target system. | String | Yes |
| `authentication` | The authentication method to use when connecting to the target system. | Authentication Object | No |
| `policies` | Array of policy identifiers to apply to this connection. | Array\[String\] | No |

You can attach policies to connection objects by referencing their policy identifier. See the [policies object](#policies) for field definitions.

#### **Policies**  {#policies}

policies

This section describes reusable, composable policy bindings and definitions that govern specific behaviors or constraints related to connections. You must reference the policy identifier to attach a policy. 

For more information about applying policies via Flex Gateway in Local Mode, see:

* [Securing Agent Interactions with Flex Gateway](https://docs.mulesoft.com/gateway/latest/flex-agent-secure)  
* [Applying Policies in Local Mode](https://docs.mulesoft.com/gateway/latest/flex-gateway-secure-local)

**Example**

| `context:   policies:     myRateLimitPolicy:       ref:         name: myReferenceObject       configuration:         - rate: 100           unit: minute` |
| :---- |

This section has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| ref | Reference to a specific policy definition. | Object | Yes |
| ref.name | Name of the reference definition | String | Yes |
| configuration | Policy-specific configuration (structure defined by the referenced policy definition). | Object | Yes |

#### **Authentication** 

authentication

This section defines the authentication schemes used by the agent network for authenticating when making outbound requests.

##### Basic Client Authentication

HTTP Basic authentication with username and password.

**Example**

| `kind: basic username: my-username password: my-password headerName: Authorization` |
| :---- |

The basic object has these properties.

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| kind | The authentication method type. | String | basic | Yes |
| username | Username for authentication. | String | String | Yes |
| password | Password for authentication. | String | String | Yes |
| headerName | The name of the HTTP used for basic credentials. If not specified, Authorization is used. | String | String. Defaults to Authorization. | No |

##### API Key Client Authentication

**Example**

| `kind: apiKey apiKey: my-api-key-123 headerName: Authorization` |
| :---- |

An API key passed in a header has these properties.

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| kind | The authentication method type. | String | apiKey | Yes |
| headerName | The name of the HTTP header that carries the API key. If not specified, Authorization is used. | String | String. Defaults to Authorization. | Yes |
| apiKey | The header or parameter name for the API key. | String | String | No |

##### API Key Client Credentials Client Authentication

API key client credentials with client ID and client secret objects.

**Example**

| `type: apikey-client-credentials client-id:   value: my-client-id   name: client_id client-secret:   value: my-client-secret   name: client_secret` |
| :---- |

The API key client credentials object has these properties.

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| kind | The authentication method type. | String | apikey-client-credentials | Yes |
| clientId | Description of the client ID to be used. | Object | Object with value and optional name (default header name is client\_id). | Yes |
| clientId.value | The value for the client ID. | String | String | Yes |
| clientId.name | The header or parameter name. | String | String | No |
| clientSecret | Description of the client secret to be used. | Object | Object with value and optional name (default header name is client\_secret). | No |
| clientSecret.value | The value for the client secret. | String | String | Yes |
| clienSecret.name | The header or parameter name. | String | String | No |

##### OAuth 2.0 Client Credentials Grant Client Authentication

OAuth 2.0 authentication using the Client Credentials Grant Type. Allows agents to obtain access tokens using a client ID and client secret from the token provider.

**Example**

| `kind: oauth2-client-credentials clientId: my-client-id clientSecret: my-client-secret token:   url: https://my-oauth2-provider.com/token   timeout: 15   bodyEncoding: form scopes:   - my.custom.scope   - another.scope` |
| :---- |

The oauth2-client-credentials object has these properties.

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| kind | The authentication method type. | String | oauth2-client-credentials | Yes |
| clientId | The client ID to authenticate with the OAuth2 provider. | String | String | Yes |
| clientSecret | The client secret used with client ID. | String | String | Yes |
| token | Configuration for how to fetch the token. | Object | Token object  | Yes |
| token.url | The URL of the token provider (token endpoint). | String | Valid URL string | Yes |
| token.timeout | Time in seconds to wait for the token service to respond. | Number | Any number | No |
| token.bodyEncoding | Content encoding for the request body. (`form = application/x-www-form-urlencoded`, `json = application/json`) | String | `form` or `json` | No |
| scopes | Array of scopes to request during token retrieval. | Array | Array of scope strings. Defaults to \[\] | No |

##### In-Task Authorization Code

Use in-task authorization code when the connection needs secondary credentials obtained during a task using the OAuth 2.0 Authorization Code flow. OAuth2 tokens are extracted from message data and injected into the Authorization header for upstream calls. This supports step-up or in-task authentication (for example, when a user must re-authenticate for a sensitive action). For more information about the associated policy, see [policies-outbound-a2a-intask-authorization-code](http://policies-outbound-a2a-intask-authorization-code.adoc).

**Example**

| `authentication:   kind: in-task-authorization-code   secondaryAuthProvider: providerName   authorizationEndpoint: https://oauth.provider.com/authorize   tokenEndpoint: https://oauth.provider.com/token   scopes: Read   redirectUri: https://oauth.provider.com/callback   responseType: code   tokenAudience: https://api.example.com/agents/my-agent   codeChallengeMethod: S256   bodyEncoding: form   challengeResponseStatusCode: 200    tokenTimeout: 300`  |
| :---- |

The in-task-authorization-code authentication has these properties.

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| kind | Authentication type. | String | in-task-authorization-code | Yes |
| authorizationEndpoint | OAuth2 authorization endpoint URL. Used to generate the authentication challenge. | String | Valid URL | Yes |
| tokenEndpoint | OAuth2 token endpoint URL. Used to generate the authentication challenge. | String | Valid URL | Yes |
| scopes | OAuth2 scopes required for step-up authentication. | String | Space- or comma-separated scope list (for example, openid profile email) | Yes |
| redirectUri | OAuth2 redirect URI the client uses in the authorization flow. | String | Valid URI | Yes |
| secondaryAuthProvider | Name of the IdP (for example, okta, auth0). Informational only, for the authentication card. | String | Any string | No |
| responseType | OAuth2 response type. | String | Typically code. Default: code | No |
| codeChallengeMethod | PKCE code challenge method. | String | Typically S256. Default: S256 | No |
| tokenAudience | Intended recipient of the token (for example, agent1 or API URL). | String | Any string | No |
| bodyEncoding | Encoding for the token request body. | String | form, json. Default: form | No |
| tokenTimeout | Timeout in seconds for token requests. | Integer | Positive integer. Default: 300 | No |
| challengeResponseStatusCode | HTTP status code returned for auth-required challenge responses. Typically 200 for JSON-RPC compatibility. | Integer | HTTP status code. Default: 200 | No |

##### OAuth 2.0 OBO Credential Injection

This authentication type supports OAuth 2.0 Token Exchange and Microsoft Entra ID On-Behalf-Of protocols. For more information about the associated policy, see [policies-outbound-oauth-obo](http://policies-outbound-oauth-obo.adoc).

**Using OAuth 2.0 Token Exchange**

| `authentication:   kind: oauth2-obo   flow: oauth2-token-exchange   tokenEndpoint: https://oauth.provider.com/token   clientId: clientId   clientSecret: clientSecret   targetType: audience   targetValue: https://api.example.com/agents/my-agent   scope: Read #optional, OAuth 2.0 scope to request. Required for Microsoft Entra OBO (for example, api://downstream-client-id/.default). Optional for OAuth 2.0 Token Exchange (RFC 8693).   timeout: 5000` |
| :---- |

**Using Microsoft Entra ID On-Behalf-Of**

| `authentication:   kind: oauth2-obo   flow: microsoft-entra-obo   tokenEndpoint: https://oauth.provider.com/token   clientId: clientId   clientSecret: clientSecret   scope: api://downstream-client-id/.default   timeout: 5000`  |
| :---- |

The `oauth2-obo` authentication has these properties.

| Parameter | Description | Type | Valid Values | Required |
| :---- | :---- | :---- | :---- | :---- |
| kind | Authentication type. | String | oauth2-obo | Yes |
| flow | Token exchange flow type. | String | oauth2-token-exchange, microsoft-entra-obo | Yes |
| clientId | OAuth2 client ID for token exchange. | String | String | Yes |
| clientSecret | OAuth2 client secret for token exchange. | String | String | Yes |
| tokenEndpoint | OAuth2 token endpoint URL for token exchange. | String | Valid URL | Yes |
| targetType | Parameter type for specifying the target service (audience for logical name, resource for physical URI). Used for OAuth 2.0 Token Exchange. | String | audience, resource. Default: audience | No |
| targetValue | Target audience URI or resource URI for the exchanged token. Required for OAuth 2.0 Token Exchange. | String | Valid URI | Required when using oauth2-token-exchange with a target |
| scope | OAuth scope to request. Required for Microsoft Entra OBO (for example, api://downstream-client-id/.default). Optional for OAuth 2.0 Token Exchange. | String | String | Required for microsoft-entra-obo |
| timeout | Timeout for token exchange requests in milliseconds. | Integer | Positive integer. Default: 10000 | No |
| cibaEnabled | When `true`, it uses Client Initiated Backchannel Authentication (CIBA) instead of standard OBO token exchange. Only valid when `flow` is `oauth2-token-exchange`. | Boolen | `true` or `false`. Default: `false` | No |
| cibaEndpoint | Backchannel authorization (`bc-authorize`) endpoint URL for CIBA. | String | Valid URL  | No |
| cibaLoginHintClaim | JWT claim from the subject token to send as the CIBA `login_hint` (for example, `email,` `sub`). | String | String. Default: `email` | No |
| cibaBindingMessage | Optional message shown on the user’s authentication device during CIBA approval. | String | String | No |

### **Brokers Section** {#brokers-section}

Use this section to define brokers that orchestrate and control the flow of agent and tool invocations. Each broker object maps to an `.agent` file. You can have multiple Agent Script files  in your Brokers folder.

**Example**

| `brokers:   customerServiceAgent:     kind: AgentScript     implementation: ./brokers/customer-service.agent     interfaces:       a2a:         card:             ...         policies:           inbound:             - ref:                 name: rate-limit-policy   billingAgent:     kind: AgentScript     implementation: ./brokers/billing-handler.agent     interfaces:       a2a:         card:           ....` |
| :---- |

The `brokers` section has these properties.

| Parameter | Description | Type | Required | Values |
| :---- | :---- | :---- | :---- | :---- |
| `_(broker id)_` | Named broker definition. | Object | No | Keys: identifiers matching ^\[a-zA-Z\_\]\[a-zA-Z0-9\_.-\]\*$. Values: broker entity (BrokerEntity in agent\_network\_v2.json) |
| `kind` | The type of agent implementation. | String | Yes | `AgentScript` |
| `implementation` | The path to the Agent Script implementation file. | String | Yes | — |
| `interfaces` | Exposed interfaces for this broker. | Object | Yes | `a2a` |

#### **interfaces.a2a**

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `card` | The A2A agent card. See [A2A Card](#a2a-card). | AgentCard  | No |
| `policies` | Inbound and outbound policy binding lists for this interface. | Object | No |
| `policies.inbound` | Policy bindings for inbound traffic. | Array | No |
| `policies.outbound` | Policy bindings for outbound traffic. | Array | No |

Each inbound or outbound policy requires either a reference or inline policy binding.

##### Policy Binding Reference

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `ref.name` | Name of the declared policy binding. See [Policies](#policies). | String | Yes |

##### Inline Policy Binding

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `policy` | The policy to be applied. | Object | Yes |
| `policy.ref` | Reference to the policy definition. | PolicyRef See [Policies](#policies). | Yes |
| `policy.configuration` | Policy-specific configuration. | Object | Yes |

### **Expressions Format** {#expressions-format}

Use Python expressions to resolve or calculate values that are only available at runtime.

Enclose expressions using the {{}} evaluation wrapper. You can also use multiline statements using the [appropriate YAML syntax](https://yaml.org/spec/1.2.2/#54-line-break-characters).

**Example** 

| `- reasoning:     id: welcome-message     description: "Send a welcome message to the candidate."     llm: my-primary-llm     prompt: "Extract the email from {{input}} and send a greeting email to the user saying their onboarding process has started"     tools:       - ref: my-email-tool` |
| :---- |

Expressions have these properties. 

| Variable Name | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| input | Payload or parameters provided as input to the node. | Type defined by Graph | Yes |
| variables | A mutable key-value store for variables scoped to the execution of the graph instance. | Dict\[string, any\] | Yes |
| {node-ids} | Read-only mapping of all node definitions available in the current graph, indexed by node id. | Dict\[string, Object\] | Yes |

{node-ids} have these properties. 

| Variable Name | Type | Description | Required |
| :---- | :---- | :---- | :---- |
| input | Type defined by Graph | Input to the node as it happened. | Yes |
| output | Type defined by Graph | Output provided by the node as it happened | Yes |

### **Exchange JSON File**  {#exchange-json-file}

All agent network projects have an `exchange.json` file. This file contains asset metadata available in Anypoint Exchange after publishing your agent network assets.

**Example** 

| `{   "main": "agent-network.yaml",   "name": "Employee Onboarding Network",   "classifier": "agentic-network",   "organizationId": "85de5a54-1f33-4ea4-a1bf-8a65bc409179",   "descriptorVersion": "1.0.0",   "tags": [],   "groupId": "85de5a54-1f33-4ea4-a1bf-8a65bc409179",   "assetId": "employee-onboarding-network",   "version": "1.0.5",   "dependencies": [     {       "groupId": "85de5a54-1f33-4ea4-a1bf-8a65bc409179",       "assetId": "hr-agent",       "version": "1.0.21",       "classifier": "agent-metadata",       "packaging": "zip"     }   ],   "metadata": {     "variables": {       "openai": {         "clientId": {           "description": "OpenAI LLM Client ID",           "default": ""         },         "clientSecret": {           "description": "OpenAI LLM Client Secret",           "default": "",           "secret": true         },         "url": {           "description": "OpenAI URL",           "default": "",           "secret": false         }       }     }   } }` |
| :---- |

These key-value pairs are important for your agent network configuration:

* groupId: ID of the Anypoint business group that owns your agent network and all the assets derived from it.  
* assetId: Unique identifier for the agent network project.  
* dependencies: Existing assets that this network needs to reference.  
* variables: Nested in the metadata section, defines all the variables whose values shouldn't be hardcoded in the `agent-network.yaml` file. Enter these values when publishing the agent network. Each variable has a description value, a default value, and a secret value that indicates whether it's treated as sensitive or not.

## Agent Script Reference {#agent-script-reference}

After you configure the assets and other elements of your agent network project, you build the rest of the workflow using Agent Script. Agent Script enables you to build predictable, context-aware agent workflows that don't rely solely on interpretation by an LLM. 

[Agent Script Structure](#agent-script-structure)

[Node Types](#node-types)

[A2A Namespace Functions](#a2a-namespace-functions)

[Built-in Functions](#built-in-functions)

[Node Outputs](#node-outputs)

[Node Expressions and References](#node-expressions-and-references)

### **Agent Script Structure** {#agent-script-structure}

The following explains settings and configurations specific to MuleSoft agent network projects. To learn more about Agent Script, see the [Agent Script documentation](https://developer.salesforce.com/docs/ai/agentforce/guide/agent-script.html). 

#### **Dialect Referencing and Versioning**

AgentScript files contain a header specifying the dialect and a version binding. SEMVER Major and minor are used for fixing to a specific dialect version.

The dialect header specifies that the script is strictly bound to a specific version or later of the AGENTFABRIC dialect. Deploying an agent to a runtime that doesn't support this version results in an error.

* Using major.minor (for example, `AGENTFABRIC=1.1`) binds to version 1.1 or later  
* Using major only (for example, `AGENTFABRIC=1`) references the latest version within that major version

**Example**

| `# @dialect: AGENTFABRIC=1.0-BETA` |
| :---- |

#### **System Section**

This section defines the `instructions` attribute, which acts as a default system prompt that will be used whenever an agentic node doesn't define a `system.instructions` of its own.

**Example**

| `system:   instructions: "You are the onboarding agent"` |
| :---- |

The system section has these parameters.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `instructions` | Default system prompt used when an agentic node doesn't define its own `system.instructions`. | String | Yes |

#### **Agent Config Section** 

The config section is the standard Agent Script config section, with the addition of the optional `default_llm` field. This section defines metadata and default settings for the agent.

**Example**

| `config:   agent_name: "employee-onboarding"   label: "Employee Onboarding Agent"   description: "An Agent that performs employee onboarding"` |
| :---- |

The config section has these parameters.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `agent_name` | The name identifier for the agent. | string | \- |
| `label` | A human-readable display name for the agent. | string | \- |
| `description` | A description of what the agent does. | string | \- |
| `default_llm` | Specifies a default LLM to be used on all agentic nodes that don't specify otherwise. | @llm reference See [LLM](#llm-section) | No |

#### **LLM Section** {#llm-section}

The `llm` element is where you define the LLMs to use for reasoning and generation. Each `target` must use the `llm://` URI scheme so the runtime binds to the correct governed connection.

**Example**

| `llm:    open-api-llm:     target: "llm://open_ai_connection"     kind: "OpenAI"     model: "gpt5-mini"     reasoning_effort: "LOW"   gemini-llm:     target: "llm://gemini_connection"     kind: "Gemini"     model: "gemini-3-flash-preview"     thinking_level: "HIGH"     top_p: 0.3` |
| :---- |

##### LLM Configuration: OpenAI

The OpenAI configuration has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `target` | Governed LLM connection as a URI; must use the `llm://` scheme | URI (`llm://…`)  | Yes |
| `kind` | Discriminator for the LLM provider; selects which provider-specific attributes apply | String, `OpenAI` | Yes |
| `model` | The name of the model to use | String | Yes |
| `reasoning_effort` | Constrains effort on reasoning for reasoning models. gpt-5.1 defaults to NONE, previous ones default to MEDIUM | enum\['NONE', 'MINIMAL', 'LOW', 'MEDIUM', 'HIGH'\] | No |
| `temperature` | Controls randomness in the output | number | No |
| `top_p` | Nucleus sampling parameter | number | No |
| `top_logprobs` | Number of most likely tokens to return at each position | integer | No |
| `max_output_tokens` | Maximum number of tokens to generate | integer | No |

##### LLM Configuration: Gemini

The Gemini configuration has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `target` | Governed LLM connection as a URI; must use the `llm://` scheme | URI (`llm://…`)  | Yes |
| `kind` | Discriminator for the LLM provider; selects which provider-specific attributes apply | String, `Gemini` | Yes |
| `model` | The name of the model to use | String | Yes |
| `thinking_level` | The level of thoughts tokens that the model should generate | Enum\['LOW', 'HIGH'\] | No |
| `thinking_budget` | Indicates the thinking budget in tokens. 0 is DISABLED. \-1 is AUTOMATIC. The default values and allowed ranges are model dependent | Number | No |
| `temperature` | Controls the degree of randomness in token selection. Lower temperatures are good for prompts that require a less open-ended or creative response, while higher temperatures can lead to more diverse or creative results | Number | No |
| `top_p` | Tokens are selected from the most to least probable until the sum of their probabilities equals this value. Use a lower value for less random responses and a higher value for more random responses | Number | No |
| `response_logprobs` | Whether to return the log probabilities of the tokens that were chosen by the model at each step | Boolean | No |
| `max_output_tokens` | Maximum number of tokens that can be generated in the response | Integer | No |

#### **Action Definitions**

You define A2A and MCP actions in Agent Script under the top-level `actions` block. Each action `target` uses a URI whose scheme is the underlying protocol (for example `a2a://` or `mcp://`), so the runtime can route the connection correctly.

##### A2A Actions

A2A actions execute the `message/send` A2A method and do not specify inputs or outputs.

**Example**

| `actions:   hr_agent:     target: "a2a://hr_agent_connection"     kind: "a2a:send_message"` |
| :---- |

A2A actions have these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `target` | Governed A2A connection as a URI; must use the `a2a://` scheme | URI (`a2a://…`)  | Yes |
| `kind` | Indicates that this executes the message/send A2A method. | "a2a:send\_message" | Yes |

##### MCP Actions

MCP actions invoke Model Context Protocol actions with optional input binding.

**Example**

| `actions:   send_slack_message:     target: "mcp://slack_mcp_connection"     kind: "mcp:tool"     tool_name: "send-message"     inputs:       channel: string = "my-default-channel"       message: string` |
| :---- |

MCP actions have these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `target` | Governed MCP connection as a URI; must use the `mcp://` scheme | URI (`mcp://…`)  | Yes |
| `kind` | Constant indicating this will invoke an MCP tool | "mcp:tool" | Yes |
| `tool_name` | The name of the tool to call | String | Yes |
| `inputs` | Define bindable arguments. Input arguments provided are not exhaustive. The tool will auto-discover additional arguments and consider them in slot filling mode. | Object | No |

####  **A2A Trigger** 

Triggers reference one of the interfaces defined for a broker in the agent network. Each broker must have one–and only one–trigger per each interface declared in its agent network.

The A2A trigger reacts to send/message methods and automatically manages the task history, context ID and task IDs. The trigger also responds to various A2A protocol methods.

* [Get Task](https://a2a-protocol.org/latest/specification/#313-get-task)  
* [List Tasks](https://a2a-protocol.org/latest/specification/#314-list-tasks)  
* [Cancel Task](https://a2a-protocol.org/latest/specification/#315-cancel-task)  
* [Subscribe to Task](https://a2a-protocol.org/latest/specification/#316-subscribe-to-task)  
* [Create Push Notification Config](https://a2a-protocol.org/latest/specification/#317-create-push-notification-config)  
* [Get Push Notification Config](https://a2a-protocol.org/latest/specification/#318-get-push-notification-config)  
* [List Push Notification Config](https://a2a-protocol.org/latest/specification/#319-list-push-notification-configs)  
* [Delete Push Notification Config](https://a2a-protocol.org/latest/specification/#3110-delete-push-notification-config)  
* [Get Extended Agent Card](https://a2a-protocol.org/latest/specification/#3111-get-extended-agent-card)

**Example**

| `trigger employeeOnboardingTrigger:   kind: "a2a"   target: "brokers://employee-onboarding/a2a"   on_message: -> transition to @orchestrator.hrSystemOnboard` |
| :---- |

The A2A trigger has these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `kind` | Value that indicates this is an A2A trigger. | `"a2a"` | Yes |
| `target` | Broker interface entry point. Must use the `brokers://` URI form: `brokers://<brokerName>/<interfaceName>`  | URI (`brokers://…`) | Yes |
| `on_message` | Procedure that executes when the A2A interface receives a new `message/send` request. Must define a transition to the workflow's initial node. | Procedure | Yes |

### **Node Types** {#node-types}

Agent network and Agent Script support these node types. 

#### **Subagent Node**

This node defines a generic agent loop node, made of a prompt and a set of actions. Because it can use actions and supports human-in-the-loop flows, this node is ideal for implementing patterns like classification, semantic routing, or LLM reasoning.

**Example**

|  `- subagent profile-extractor:     description: "Extracts structured user profile data from text"     reasoning:       instructions: ->          Extract the following information from the user's message: {!@request.payload.message.parts[0].text}       outputs:         properties:           name:             type: "string"             description: "Full name of the person"             minLength: 1           email:             type: "string"             description: "Email address"             pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"           age:             type: "integer"             description: "Person's age"             minimum: 0             maximum: 150           preferences:             type: "object"             description: "User preferences"             properties:               newsletter:                 type: "boolean"                 description: "Whether user wants newsletter"                 default: "false"               category:                 type: "string"                 description: "Preferred category"                 enum:                   - "tech"                   - "business"                   - "sports"           tags:             type: "array"             description: "Interest tags"             items:               type: "string"             minItems: 1             maxItems: 10     on_exit: ->        transition to @orchestrator.process_profile` |
| :---- |

The subagent node has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `id` | The node identifier, defined next to the node type. | String | Yes |
| `label` | An optional short, human-readable display name for the node. | String | No |
| `description` | A CommonMark string providing a description of the node. | String | No |
| `on_exit` | A procedure that executes when the node execution finishes. | Procedure | Yes |
| `llm` | Overrides the default LLM setting | @llm reference See [LLM Section](#llm-section) | No |
| `system.instructions` | Overrides the global `system.instructions` at the file root level | String | No |
| `reasoning.instructions` | Session-specific query or instructions for this particular node, typically containing user provided or user related context | String | Yes |
| `reasoning.actions` | The available actions | Array\[actions\] | No |
| `reasoning.outputs` | Schema definition describing the expected structure of the agent's output | Outputs See [Node Outputs](#node-outputs) | No |
| `reasoning.max_number_of_loops` | The maximum number of loops an execution can take. Useful for keeping it from running too long and consuming too many tokens. Default: 25 | Integer | No |
| `outputs` | A schema definition for the agentic output. | Object See [Node Outputs](#node-outputs) | No |

#### **Orchestrator Node**

The orchestrator node is a specialization of the subagent node used for orchestrating multiple agents and MCP servers to achieve a specified goal. It is optimized for multi-agent orchestration. Use this node type for workflows that need to call multiple external agents or actions to achieve a goal.

**Example**

|   `orchestrator flight-booking-agent:     description: books flights by looking for the best offer across approved partners     system:       instructions: |         You are a flight booking agent.            The process for flight booking is:           1. Ask the user for a destination and travel dates and present them with matching alternatives using available actions.           2. Allow the user to change or refine the search           3. Once the user selects a flight, book it using the concur agent tool     reasoning:       instructions: ->          @request.payload.message.parts[0].text       actions:         search-flight: @actions.search-flight           with companyId = @variables.companyId              get-flight-info: @actions.get-flight-info              concur: @actions.concur-agent           with http_headers = {"Authorization": @request.headers["Authorization"]}     outputs:       properties:         flightNumber:           type: "string"           description: "The flight identification number"         airline:           type: "string"           description: "The airline name"      max_number_of_loops: 10      task_timeout_secs: 60          on_exit: ->        transition to @executor.send_summary` |
| :---- |

The orchestrator node has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `id` | The node identifier, defined next to the node type. | String | Yes |
| `label` | An optional short, human-readable display name for the node. | String | No |
| `description` | A CommonMark string providing a description of the node. | String | No |
| `on_exit` | A procedure that executes when the node execution finishes. | Procedure | Yes |
| `llm` | Overrides the default LLM setting | @llm reference See [LLM Section](#llm-section) | No |
| `system.instructions` | Overrides the global `system.instructions` at the file root level | String | No |
| `reasoning.instructions` | Session-specific query or instructions for this particular node, typically containing user provided or user related context | String | Yes |
| `reasoning.actions` | The available actions | Array\[actions\] | No |
| `reasoning.outputs` | Schema definition describing the expected structure of the agent's output | Outputs See [Node Outputs](#node-outputs) | No |
| `reasoning.max_number_of_loops` | The maximum number of loops an execution can take. Useful for keeping it from running too long and consuming too many tokens. Default: 25 | Integer | No |
| `outputs` | A schema definition for the agentic output. | Object See [Node Outputs](#node-outputs) | No |

#### **Generator Node**

The generator node calls an LLM to generate text. It is not an agent loop, and it does not support human-in-the-loop learning or other actions. It performs exactly one LLM call. Use this node for summarization, formatting, or templated text generation.

**Example**

| `generator summarize-report:     description: "Generate a one-paragraph summary of the report."     prompt: "Summarize the following in one paragraph: {!@variables.report}"     on_exit: ->        transition to ...` |
| :---- |

The generator node has these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `id` | The node identifier, defined next to the node type. | String | Yes |
| `label` | An optional short, human-readable display name for the node. | String | No |
| `description` | A CommonMark string providing a description of the node. | String | No |
| `on_exit` | A procedure that executes when the node execution finishes. | Procedure | Yes |
| `llm` | A reference to the LLM connection. | @llm reference See [LLM Section](#llm-section) | No |
| `system.instructions` | Overrides the global `system.instructions` at the file root level for this generator node. | String | No |
| `prompt` | Session-specific query or instructions for this particular node, typically containing user provided or user related context. | String | Yes |
| `outputs` | A schema definition for the agentic output. | Object See [Node Outputs](#node-outputs) | No |

#### **Executor Node**

The executor node is used to execute a set of Agent Script statements, primarily for setting variables or deterministic tool invocations. Use this node to set variables or call actions with known or fixed arguments.

**Example**

| `executor sendHrSlackUpdate:   do: ->     run @actions.send_slack_message     with text= @generator.generate-hr-slack-update-message.output     with channel_id= "my-onboarding-channel-id"   on_exit: ->      transition to @router.countrySwitch` |
| :---- |

The executor node has these properties. 

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `id` | The node identifier, defined next to the node type. | String | Yes |
| `label` | An optional short, human-readable display name for the node. | String | No |
| `description` | A CommonMark string providing a description of the node. | String | No |
| `on_exit` | A procedure that executes when the node execution finishes. | Procedure | Yes |
| `do` | Agent Script statements to execute | procedure | Yes |

#### **Router Node**

The router node performs dynamic transitions based on deterministic conditions. This node does not support transition to in its on\_exit attribute. Use this node for branching based on structured output from a previous node.

**Example**

| `router countryRouter:   routes:     - target: @orchestrator.argentinaOnboard       when: @orchestrator.hrSystemOnboard.output.country  == "ARG"       label: "Argentina"     - target: @orchestrator.usOnboard       when: @orchestrator.hrSystemOnboard.output.country  == "USA"       label: "USA"   otherwise:     target: @echo.invalidCountryResponse` |
| :---- |

The router node has these properties.

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `id` | The node identifier, defined next to the node type. | String | Yes |
| `label` | An optional short, human-readable display name for the node. | String | No |
| `description` | A CommonMark string providing a description of the node. | String | No |
| `routes` | An array of condition and target pairs, plus an optional label field for UI. Must define at least one route. Each route contains: `target`, `when`, and optional `label`. | Array | Yes |
| `otherwise` | Defines a default transition when no route condition matches. Contains: `target`. | Object | Yes with `routes` |

#### **Echo Node**

The echo node sends a response back to the client. The number of responses depends on the trigger interface and its configuration. Use this node for the end of a workflow, or anytime you want to emit a response. Currently only supports `a2a:response` (non-streaming). 

**Example**

| `echo a2a_response:   kind: "a2a:response"   task: a2a.task({     state: "completed",     message: a2a.message(           {             messageId: uuid(),             parts:[               a2a.textPart("You have been onboarded!your employee ID is" +@orchestrator.hrSystemOnboard.output.employeeId)             ]           }),     artifacts: a2a.parts(*@variables.artifacts),     metadata:None     })` |
| :---- |

| Parameter | Description | Type | Required |
| :---- | :---- | :---- | :---- |
| `id` | The node identifier, defined next to the node type. | String | Yes |
| `label` | An optional short, human-readable display name for the node. | String | No |
| `description` | A CommonMark string providing a description of the node. | String | No |
| `on_exit` | A procedure that executes when the node execution finishes. | Procedure | No, an echo node can be a terminal node. |
| `kind` | Discriminator for the response type. Must be "a2a:response". | String | Yes |
| `task` | A Task object as defined in the A2A specification. The `id`, `contextId` and `history` attributes are automatically populated by the trigger | Task object | Yes |

### **A2A Namespace Functions** {#a2a-namespace-functions}

The `a2a` namespace provides a set of functions that support A2A Task object creation. Do not prefix these functions with `@` as it’s  reserved for references such as `@variables`, `@actions`, `@request`, and `@orchestrator.<nodeId>`.

### 

| Function | Description | Input Arguments | Output | Example |
| :---- | :---- | :---- | :---- | :---- |
| `a2a.task` | Builds an A2A Task response object | `state: str` (required) `message` (optional, from `a2a.message`) `artifacts: list` (optional, from `a2a.artifact`) `metadata: dict` (optional) | `dict` (Task) | `a2a.task("completed", message=a2a.message(...), artifacts=[a2a.artifact(...)])` |
| `a2a.message` | Builds an A2A Message object | `parts: list` (required, from `a2a.textPart/a2a.dataPart/a2a.filePart`) `role: str` (optional, default: "agent") `metadata: dict` (optional) | `dict` (Message) | `a2a.message([{messageId: uuid(), parts: [a2a.textPart("Hello")]}])` |
| `a2a.textPart` | Builds a TextPart object (kind: "text") | `text: str` (required) `metadata: dict` (optional) | `dict` (TextPart) | `` a2a.textPart("Employee ID: {!@orchestrator.employee.id}") `a2a.textPart("Status: Complete", metadata={priority: "high"})` `` |
| `a2a.dataPart` | Builds a DataPart object (kind: "data") | `data: dict` (required) `metadata: dict` (optional) | `dict` (DataPart) | `` a2a.dataPart({employeeId: "E123", department: "Engineering"}) `a2a.dataPart(@orchestrator.result.output)` `` |
| `a2a.filePart` | Builds a FilePart object (kind: "file") | `uri: str` (optional, required if bytes not provided) `bytes: str` (optional, base64-encoded) `name: str` (optional) `mime_type: str` (optional) `metadata: dict` (optional) | `dict` (FilePart) | `a2a.filePart(uri="https://example.com/report.pdf", name="report.pdf", mime_type="application/pdf") a2a.filePart(bytes="SGVsbG8gV29ybGQ=", name="data.txt")` |
| `a2a.artifact` | Builds an A2A Artifact object with auto-generated ID | `parts: list` (required, from `a2a.textPart/a2a.dataPart/a2a.filePart`)  `artifact_id: str` (optional, auto-generated)  `name: str` (optional)  `description: str` (optional) `metadata: dict` (optional) | `dict` (Artifact) | `` a2a.artifact([a2a.dataPart(...)], name="Results", description="Analysis results") `a2a.artifact([a2a.filePart(...)], artifact_id="custom-id")` `` |
| `a2a.parts` | Collects multiple items into a list for composing parts/artifacts arrays | `*args: Any` (variable arguments) | `list` | `` a2a.parts(*@variables.artifacts) `a2a.parts(a2a.textPart("Part 1"), a2a.dataPart({key: "value"}))` `` |

### 

Usage notes:

* Functions are designed to be composed: `a2a.task` accepts `messages` created by `a2a.message`, which accepts `parts` created by `a2a.textPart`/`a2a.dataPart`/`a2a.filePart.`  
* `a2a.filePart` requires either `uri` OR `bytes` (base64-encoded), but not both  
* `a2a.artifact` auto-generates `artifactId` if it’s not provided.  
* Use `a2a.parts` with the `*` operator to unpack arrays, for example, `a2a.parts(*@variables.artifacts)`.  
* All `metadata` parameters are optional and accept arbitrary dictionaries.  
* For tasks, it's not necessary to define the `id`, `contextId` and `history` attributes, those are automatically populated by the trigger.

### **Built-in Functions** {#built-in-functions}

Use these functions in expressions alongside references and interpolations. They include time and ID helpers (`now`, `uuid`), string utilities (`strip`, `startswith`, and `endswith`), JSON parsing (`parse_json`), and common numeric helpers (`abs`, `round`, and `sum`) for deterministic math-style logic without calling external tools.

| Function | Description | Input arguments | Output | Example |
| :---- | :---- | :---- | :---- | :---- |
| `now` | Current UTC time in ISO 8601 format | None | String (ISO 8601\) | `now()` |
| `uuid` | Random UUID v4 | None | String (UUID) | `uuid()` |
| `strip` | Removes leading/trailing characters from a string | String, optional chars (default: whitespace) | String | `strip(" hello ") → "hello"` |
| `startswith` | Whether a string starts with a prefix | String, prefix | Boolean | `startswith("hello world", "hello")` |
| `endswith` | Whether a string ends with a suffix | String, suffix | Boolean | `endswith("report.pdf", ".pdf")` |
| `abs` | Absolute value | Number | Number | `abs(-42)` |
| `round` | Round to optional digit count | Number, optional `ndigits` | Number | `round(3.14159, 2)` |
| `sum` | Sum of a numeric list | List | Number | `sum([10, 20, 30])` |
| `parse_json` | Parse a JSON string | String (valid JSON) | Object or array | `parse_json('{"key": "value"}')` |

### **Node Outputs** {#node-outputs}

Use the `outputs` field to define the expected shape of the agent's response using a schema notation similar to a JSON schema. When provided, the agent produces output matching the defined structure for downstream parsing and processing.

Each property maps to a field in the agent's output. The following types are supported.

**Note:** Advanced JSON schema features are not supported, so do not copy patterns from generic JSON Schema tutorials unless they match what is documented here.

| Type | Description |
| :---- | :---- |
| String | Text values with optional constraints like `pattern` (regex), `minLength`, `maxLength`, and `enum` (allowed values). |
| Number / Integer | Numeric values with optional constraints like `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, and `enum`. |
| Boolean | `True` or `False` (with optional default). |
| Array | Lists of items, where `items` define the schema for each element (can be any supported type). Supports `minItems` and `maxItems`. |
| Object | Nested structures with their own `properties` map. Supports a `required` array to specify mandatory fields. |

Each property definition can include:

* `type`: The data type (required).  
* `description`: A human-readable explanation of the property's purpose.  
* `default`: A default value if the property is omitted.

The `outputs` definition does not support the following:

* `additionalProperties` or similar JSON Schema extensibility flags  
* Combinators such as `anyOf`, `oneOf`, or `allOf`  
* References or shared definitions (`$ref`, `$defs`)  
* Composition beyond nested `object` / `array` structures as described above

### **Node Expressions and References** {#node-expressions-and-references}

Nodes access data from other parts of the workflow using expressions.

The following references are used.

| Prefix | Reference | Example |
| :---- | :---- | :---- |
| `@llm.` | LLM definitions | `@llm.open-api-llm` |
| `@actions.` | action definitions | `@actions.hr_agent` |
| `@request.` | Trigger request data | `@request.payload`, `@request.interface` |
| `@request.headers` | HTTP headers (case-insensitive) | `@request.headers["Authorization"]` |
| `@variables.` | Workflow variables | `@variables.companyId` |
| `@<nodeType>.<nodeId>.` | Node references | `@orchestrator.hrOnboard.output` |

#### **Accessing Node Output and Input**

Every node has `.output` (the value it produced) and `.input` (the output of whichever node transitioned into it).

**Example**

| ``@orchestrator.hrSystemOnboard.output.employeeId       # returns the `employeeId` property of the object returned by the `hrSystemOnboard` node    @generator.writeEmailContent.output     # returns the string generated by the `writeEmailContent` node`` |
| :---- |

* Use `.output` when you know exactly which upstream node you’re referencing.  
* Use `.input` when multiple nodes transition into the current one and you want to decouple it from the specific path taken.

In this example, `@generate.generate_email.input` returns whichever of node\_a/b/c actually transitioned into it.

| `node_a ──┐             │ node_b ──┼──► generate_email ──► send_email             │ node_c ──┘` |
| :---- |

#### **Setting Action Headers**

Any actions that connect to an external system often need to set custom headers. Use cases range from propagating authorization headers to adding custom correlation information.

For this, both the MCP and A2A actions automatically get an implicit optional `http_headers` parameter object type that can be used to set those:

| `actions:   my_hr_agent: @actions.hr_agent     with http_headers = {"Authorization": @request.headers["Authorization"], "X-CorrelationId": @variables.conversationId}`  |
| :---- |

#### **String Interpolation**

Use `{!expression}` to embed values inside strings.

**Example**

| `prompt: "The employee's country is {!@orchestrator.hrOnboard.output.country}"` |
| :---- |

#### **Slot Filling** 

Use slot filling (`...`) to tell an LLM to figure out a value.

**Example**

| `actions:   send_message: @actions.send_slack_message     with message = ...    # LLM decides the message content` |
| :---- |

#### **Tool Binding at the Node Level**

When you reference a tool inside a node, you can fix, default, or slot-fill its arguments using `with`.

**Example**

| `actions:   # All arguments via slot filling (LLM decides everything)   sendToDefault: @actions.send_slack   # Fix the channel, LLM fills the message   sendToFixed: @actions.send_slack     with channel = "agent-fabric"   # Fix everything — fully deterministic   fullyDeterministic: @actions.send_slack     with message = @variables.calculatedMessage     with channel = @variables.channelId` |
| :---- |

# Appendix: Building an IT Investigation Broker {#appendix:-building-an-it-investigation-broker}

In this example, the goal is to use MuleSoft Vibes to build an IT Investigation agent that triages incoming support tickets, escalates critical issues, resolves common problems through cross-platform investigation, and handles licensing requests.

In each phase you interact with MuleSoft Vibes as it builds your agent network project. Here’s how it works.

## Phase 1: Input Functional Requirements

In this first phase, you define the functionality of the IT Investigation agent, which is to triage, investigate, and resolve support tickets. The agent must read incoming ticket descriptions (received via an A2A message with a Jira ticket ID), classify severity (High, Low, or too vague to classify) using an LLM's judgment, and then take the correct action. If the ticket is too vague, the agent first asks the user for clarification. High-severity tickets must be immediately routed for escalation using an Escalation MCP tool. Low-severity tickets require multi-agent investigation, using a Help Center agent and a License Procurement agent. This investigation path results in one of three deterministic outcomes: "Help Given," "License Given," or "Unresolved," the latter of which requires escalation to a human agent. Crucially, the severity classification and cross-platform investigation are open-ended, but the routing based on that classification and the final resolution must be deterministic.

## Phase 2: Asset Registration

In this phase, you work with MuleSoft Vibes to identify and register the necessary external tools and LLMs to support the agent's requirements. Since the investigation and resolution actions require four external assets, you select two A2A agents: the Help Center Agent for searching the knowledge base and the License Procurement Agent for checking software licenses. You also select two MCP tools: the Escalation MCP Server for handling high-severity cases and the Jira MCP Server for updating ticket status. For the LLMs, you select OpenAI GPT-5.4 for reasoning and orchestration tasks, such as cross-platform triage. You register Gemini 2.5 Flash for fast, one-shot summaries and response message generation.

## Phase 3: Define the Broker (Agent Script)

In this phase, MuleSoft Vibes establishes the logical workflow for the IT Investigation agent by defining a series of interconnected nodes. The graph begins with an A2A Trigger node (`ticketTrigger`) that receives the incoming ticket and Jira ID. This immediately flows into a generator node (`classifySeverity`), which uses LLM reasoning to classify the ticket as High or Low , and produces a structured output.

The deterministic `severityRouter` node then directs the process into two paths based on the classification output:

* High: An executor node calls the Escalation MCP tool.  
* Low: An orchestrator node coordinates agents and tools to investigate, producing a structured outcome (e.g., `help_given`). A second router then directs the flow based on this outcome, with all paths concluding via an Echo node.

## Phase 4: Asset Assignment to Graph

In this phase, MuleSoft Vibes assigns the registered assets to the specific nodes in the graph while adhering to the Principle of Least Privilege. 

This table shows which assets and LLMs MuleSoft Vibes mapped to which nodes. 

| Node | Type | Assets | LLM | Total Assets | Notes |
| ----- | ----- | ----- | ----- | ----- | ----- |
| `classifySeverity` | generator | (none) | Gemini 2.5 Flash (default) | 0 | Pure LLM classification (needs deep reasoning to classify ambiguous tickets). Constraint: Can only read the ticket and classify, has no actions. |
| `crossPlatformTriage` | orchestrator | Help Center A2A, License Procurement A2A, Jira MCP | OpenAI GPT 5.4 | 3 | Coordinates multi-agent investigation (needs deep reasoning). Constraint: Can investigate and update tickets, but cannot escalate. |
| `helpSummary` | generator | – | Gemini 2.5 Flash (default) | – | One-shot summary generation. |
| `licenseSummary` | generator | – | Gemini 2.5 Flash (default) | – | One-shot summary generation. |
| `escalateTicket` | executor | Escalation MCP | – | 1 | Deterministic action call (no LLM). Constraint: Can only escalate, not investigate or resolve. |
| `escalateUnresolved` | executor | Escalation MCP | – | 1 | Deterministic escalation for unresolved tickets (no LLM). Constraint: Can only escalate unresolved tickets after investigation has been exhausted. |
| `All echo nodes` | echo | – | – | – | Deterministic response formatting (no LLM). |

## Phase 5: Instruction Refinement

In this phase, MuleSoft Vibes writes and validates (with you) the precise natural language instructions for the LLM-powered nodes. The instructions for the `classifySeverity` node clearly define what constitutes High and Low severity, such as classifying a system outage as High severity. The `crossPlatformTriage` node instructions outline the required investigation steps: searching the Help Center, checking the License Procurement agent, and updating the Jira ticket. Furthermore, this instruction defines the two structured resolution outcomes the node must produce: `help_given` or `license_give`n. A rigorous Contradiction Test is executed to ensure that no node's instruction accidentally overrides the deterministic routing logic defined in the graph. All instructions are verified to align with the overall graph structure and pass the cross-node checks.

## Phase 6: Final Topology Review

In this final phase, MuleSoft Vibes provides a comprehensive validation of the fully defined and instructed agent topology before deployment. The review confirms the crucial separation of concerns, ensuring LLM reasoning is used only for classification, with all subsequent routing being deterministic. The complete separation of the three severity paths—escalation, investigation, and asking for more info—is fully validated. The topology confirms that all possible paths terminate correctly at an echo response node to communicate the outcome to the original caller. Key operational parameters were also finalized, including setting maximum loop limits (5 for orchestration, 3 for classification) to prevent infinite loops. An edge case is defined, specifying that if the orchestration node cannot determine a resolution, the outcome defaults to `help_given`.

## IT Agent Broker Diagram

The following diagram shows the workflow of the graph. 

![][image4]

## IT Agent Project Reference

The following shows the project files for the IT agent. 

**Project structure**

| `it-help-network/   agent-network.yaml   exchange.json   brokers/     it-help-investigation.agent` |
| :---- |

**exchange.json**

| `{   "main": "agent-network.yaml",   "name": "IT Help Investigation Agent Network",   "classifier": "agentic-network",   "groupId": "${organizationId}",   "assetId": "it-help-investigation-network",   "version": "1.0.0",   "description": "Triages IT support tickets, escalates critical issues, and resolves common problems through cross-platform investigation with Help Center and License Procurement agents.",   "dependencies": [],   "metadata": {     "variables": {       "helpCenterAgent": {         "url": {           "description": "Help Center Agent (Agentforce) URL",           "default": "",           "secret": false         },         "clientId": {           "description": "Help Center Agent OAuth2 Client ID",           "default": "",           "secret": false         },         "clientSecret": {           "description": "Help Center Agent OAuth2 Client Secret",           "default": "",`           `"secret": true         },         "tokenUrl": {           "description": "Help Center Agent OAuth2 Token Endpoint",           "default": "",           "secret": false         }       },       "licenseProcurementAgent": {         "url": {           "description": "License Procurement Agent (Scanned) URL",           "default": "",           "secret": false         },         "clientId": {           "description": "License Procurement Agent OAuth2 Client ID",           "default": "",           "secret": false         },         "clientSecret": {           "description": "License Procurement Agent OAuth2 Client Secret",           "default": "",           "secret": true         },         "tokenUrl": {           "description": "License Procurement Agent OAuth2 Token Endpoint",           "default": "",           "secret": false         }       },       "escalationMcp": {         "url": {           "description": "Escalation MCP Server URL",           "default": "",           "secret": false         },         "apiKey": {           "description": "Escalation MCP Server API Key",           "default": "",           "secret": true         }       },       "jiraMcp": {         "url": {           "description": "Jira MCP Server URL",           "default": "",           "secret": false         },         "clientId": {           "description": "Jira OAuth2 Client ID for OBO Token Exchange",           "default": "",           "secret": false         },         "clientSecret": {           "description": "Jira OAuth2 Client Secret for OBO Token Exchange",           "default": "",           "secret": true         },         "tokenUrl": {           "description": "Jira OAuth2 Token Endpoint for OBO Token Exchange",           "default": "",           "secret": false         },         "audience": {           "description": "Jira Target Audience URI for OBO Token Exchange",           "default": "",           "secret": false         }       },       "gemini": {         "apiKey": {           "description": "Google Gemini API Key",           "default": "",           "secret": true         }       },       "openai": {         "apiKey": {           "description": "OpenAI API Key",           "default": "",           "secret": true         }       }     }   } }` |
| :---- |

**agent-network.yaml**

|  `agentNetwork: "2.0.0" info:   label: "IT Help Investigation Agent Network"   version: v1 registry:   agents:     helpCenterAgent:       info:         label: Help Center Agent       metadata:         protocol: a2a         platform: Other         card:           a2a:             protocolVersion: 0.3.0             name: Help Center Agent             description: Searches the IT knowledge base for answers to common issues. Returns relevant articles with step-by-step instructions.             url: ${ingressgw.url}/helpCenterAgent             capabilities:               pushNotifications: false             version: 1.0.0             defaultInputModes:               - application/json               - text/plain             defaultOutputModes:               - application/json               - text/plain             skills:               - id: knowledge-search                 name: Knowledge Base Search                 description: Search for IT help articles and known solutions.                 tags:                   - knowledge-base                   - it-support                 examples:                   - How do I reset my VPN password?                   - My email is not syncing                   - How do I set up two-factor authentication?                 inputModes:                   - application/json                   - text/plain                 outputModes:                   - application/json                   - text/plain     licenseProcurementAgent:       info:         label: License Procurement Agent       metadata:         protocol: a2a         platform: Other         card:           a2a:             protocolVersion: 0.3.0             name: License Procurement Agent             description: Checks software license availability and provisions licenses for employees.             url: ${ingressgw.url}/licenseProcurementAgent             capabilities:               pushNotifications: false             version: 1.0.0             defaultInputModes:               - application/json               - text/plain             defaultOutputModes:               - application/json               - text/plain             skills:               - id: license-check                 name: License Check and Provision                 description: Check license availability and provision for a user.                 tags:                   - licensing                   - provisioning                 examples:                   - Provision a Figma license for jane.doe@company.com                   - Check if we have available GitHub Enterprise seats                   - I need access to Jira                 inputModes:                   - application/json                   - text/plain                 outputModes:                   - application/json                   - text/plain   mcps:     escalationMcp:       info:         label: Escalation MCP Server       metadata:         transport:           kind: streamableHttp           path: /mcp     jiraMcp:       info:         label: Jira MCP Server       metadata:         transport:           kind: streamableHttp           path: /mcp   llms:     gemini:       info:         label: Gemini       metadata:         platform: Gemini    openai:       info:         label: OpenAI       metadata:         platform: OpenAI context:   connections:     helpCenterAgentConnection:       kind: a2a       ref:         name: helpCenterAgent       url: ${helpCenterAgent.url}       authentication:         kind: oauth2-client-credentials         clientId: ${helpCenterAgent.clientId}         clientSecret: ${helpCenterAgent.clientSecret}         token:           url: ${helpCenterAgent.tokenUrl}     licenseProcurementAgentConnection:       kind: a2a       ref:         name: licenseProcurementAgent       url: ${licenseProcurementAgent.url}       authentication:         kind: oauth2-client-credentials         clientId: ${licenseProcurementAgent.clientId}         clientSecret: ${licenseProcurementAgent.clientSecret}         token:           url: ${licenseProcurementAgent.tokenUrl}     escalationMcpConnection:       kind: mcp       ref:         name: escalationMcp       url: ${escalationMcp.url}       authentication:         kind: apiKey         apiKey: ${escalationMcp.apiKey}     jiraMcpConnection:       kind: mcp       ref:         name: jiraMcp       url: ${jiraMcp.url}       authentication:         kind: oauth2-obo         flow: oauth2-token-exchange         clientId: ${jiraMcp.clientId}         clientSecret: ${jiraMcp.clientSecret}         tokenEndpoint: ${jiraMcp.tokenUrl}         targetType: audience         targetValue: ${jiraMcp.audience}     geminiConnection:       kind: llm       ref:         name: gemini       url: https://generativelanguage.googleapis.com       authentication:         kind: apiKey         apiKey: ${gemini.apiKey}      openaiConnection:       kind: llm       ref:         name: openai       url: https://api.openai.com/v1       authentication:         kind: apiKey         apiKey: ${openai.apiKey} brokers:   it-help-investigation:     kind: AgentScript     implementation: ./brokers/it-help-investigation.agent     interfaces:       a2a:         card:           name: IT Help Desk Broker           description: Triages IT support tickets, escalates critical issues, and resolves common problems through cross-platform investigation.           url: ${ingressgw.url}/it-help-investigation           version: 1.0.0           protocolVersion: 0.3.0           capabilities:             streaming: false             pushNotifications: true           defaultInputModes:             - text/plain           defaultOutputModes:             - text/plain           skills:             - id: ticket-triage               name: IT Ticket Triage               description: Classifies and resolves IT support tickets.               tags:                 - it-support                 - help-desk` |
| :---- |

**It-help-investigation.agent file:**

|  `# @dialect: AGENTFABRIC=0.1-BETA system:   instructions: "You are an IT Help Desk agent. You triage incoming support tickets, classify their severity, and either escalate, investigate, or request more information." config:   agent_name: "it-help-investigation"   default_llm: @llm.gemini_flash llm:   gemini_flash:     target: "llm://geminiConnection"     kind: "Gemini"     model: "gemini-2.5-flash"       openai_gpt:     target: "llm://openaiConnection"     kind: "OpenAI"     model: "gpt-5.4" # -- ACTION DEFINITIONS ------------------------------------------------------- actions:   help_center_agent:     target: "a2a://helpCenterAgentConnection"     kind: "a2a:send_message"   license_procurement_agent:     target: "a2a://licenseProcurementAgentConnection"     kind: "a2a:send_message"   escalate_ticket:     target: "mcp://escalationMcpConnection"     kind: "mcp:tool"     tool_name: "escalate"     inputs:       ticket_id: string       severity: string       reason: string       description: string   update_jira_ticket:     target: "mcp://jiraMcpConnection"     kind: "mcp:tool"     tool_name: "updateIssue"     inputs:       ticket_id: string       status: string       comment: string # -- TRIGGER ------------------------------------------------------------------- trigger ticketTrigger:   kind: "a2a"   target: "brokers://it-help-investigation/a2a"   on_message: ->     transition to @generator.classifySeverity # -- SEVERITY CLASSIFICATION --------------------------------------------------- generator classifySeverity:   description: "Classifies the severity of the support ticket."   label: "Classify Severity"   system:     instructions: |       Classify the severity of the incoming IT support ticket and extract the Jira ticket ID.       Severity Levels:       - High: System outage, security incident, or blocking issue affecting multiple users         (e.g. "VPN is down for the entire office", "Nobody in Building 3 can connect",         "unauthorized login attempts from an IP in another country")       - Low: General IT question, software request, or single-user issue         (e.g. "I forgot my VPN password", "rate limited on my Figma MCP server",         "I need access to Tableau")       If the ticket is too vague to classify confidently, default to low severity.       The ticket_id must always be a string value matching Jira's format: at least 3 characters       (the project name), a dash, and a number (e.g. "MULE-2321").       If no Jira ticket ID is provided in the input, default ticket_id to "MULE-0001".   prompt: ->     | {!@request.payload.message.parts[0].text}   outputs:     properties:       ticket_id:         type: "string"         description: "The Jira ticket ID extracted from the input (e.g. 'MULE-2321')"       severity:         type: "string"         description: "The severity level"         enum:           - "high"           - "low"       reason:         type: "string"         description: "Brief explanation of the classification"   on_exit: ->     transition to @router.severityRouter # -- SEVERITY ROUTING ---------------------------------------------------------- router severityRouter:   description: "Routes based on the classified severity."   routes:     - target: @executor.escalateTicket       when: @generator.classifySeverity.output.severity == "high"       label: "High"   otherwise:     target: @orchestrator.crossPlatformTriage # -- HIGH: ESCALATION --------------------------------------------------------- executor escalateTicket:   description: "Escalates the ticket using the Escalation MCP tool."   do: ->     run @actions.escalate_ticket       with ticket_id = @generator.classifySeverity.output.ticket_id       with severity = "high"       with reason = @generator.classifySeverity.output.reason       with description = @request.payload.message.parts[0].text   on_exit: ->     transition to @echo.escalationResponse echo escalationResponse:   kind: "a2a:response"   task: a2a.task({     state: "completed",     message: a2a.message({       messageId: uuid(),       parts: [         a2a.textPart("Ticket " + @generator.classifySeverity.output.ticket_id + " has been escalated to the on-call team due to high severity: " + @generator.classifySeverity.output.reason)       ]     }),     metadata: None   }) # -- LOW: CROSS-PLATFORM TRIAGE ----------------------------------------------- orchestrator crossPlatformTriage:   description: "Investigates the ticket using Help Center and License Procurement agents."   label: "Cross-Platform Triage"   llm: @llm.openai_gpt   system:     instructions: |       Investigate this low-severity IT support ticket.       Step 1: Search the Help Center agent for relevant articles or known solutions.       Step 2: If the issue involves software licensing, check with the License Procurement agent.       Step 3: Update the Jira ticket with your findings and resolution.       If you found an answer from the Help Center, set resolution to "help_given".       If you resolved a licensing issue, set resolution to "license_given".       If you could not find a solution or the issue requires human intervention, set resolution to "unresolved".       Always update the Jira ticket with resolution notes.   reasoning:     instructions: ->       | {!@request.payload.message.parts[0].text}     actions:       search_help: @actions.help_center_agent       check_license: @actions.license_procurement_agent       update_ticket: @actions.update_jira_ticket         with ticket_id = @generator.classifySeverity.output.ticket_id         with http_headers = {"Authorization": @request.headers.Authorization}     outputs:       properties:         resolution:           type: "string"           description: "The resolution type"           enum:             - "help_given"             - "license_given"             - "unresolved"         summary:           type: "string"           description: "Summary of the resolution and actions taken"   on_exit: ->     transition to @router.resolutionRouter # -- RESOLUTION ROUTING -------------------------------------------------------- router resolutionRouter:   description: "Routes based on the resolution type from triage."   routes:     - target: @generator.licenseSummary       when: @orchestrator.crossPlatformTriage.output.resolution == "license_given"       label: "License Given"     - target: @executor.escalateUnresolved       when: @orchestrator.crossPlatformTriage.output.resolution == "unresolved"       label: "Unresolved"   otherwise:     target: @generator.helpSummary # -- HELP GIVEN PATH ---------------------------------------------------------- generator helpSummary:   description: "Generates a summary of the help resolution."   system:     instructions: "You generate clear, friendly summaries of IT help desk resolutions."   prompt: ->     | Generate a resolution summary for the user. Original request: {!@request.payload.message.parts[0].text}. Resolution and actions taken: {!@orchestrator.crossPlatformTriage.output.summary}   on_exit: ->     transition to @echo.helpResponse echo helpResponse:   kind: "a2a:response"   task: a2a.task({     state: "completed",     message: a2a.message({       messageId: uuid(),       parts: [         a2a.textPart(@generator.helpSummary.output)       ]     }),     metadata: None   }) # -- LICENSE GIVEN PATH -------------------------------------------------------- generator licenseSummary:   description: "Generates a summary of the license provisioning."   system:     instructions: "You generate clear, friendly summaries of license provisioning actions."   prompt: ->     | Generate a license provisioning summary for the user. Original request: {!@request.payload.message.parts[0].text}. Resolution and actions taken: {!@orchestrator.crossPlatformTriage.output.summary}   on_exit: ->     transition to @echo.licenseResponse echo licenseResponse:   kind: "a2a:response"   task: a2a.task({     state: "completed",     message: a2a.message({       messageId: uuid(),       parts: [         a2a.textPart(@generator.licenseSummary.output)       ]     }),     metadata: None   }) # -- UNRESOLVED PATH ----------------------------------------------------------- executor escalateUnresolved:   description: "Escalates an unresolved low-severity ticket to a human agent."   do: ->     run @actions.escalate_ticket       with ticket_id = @generator.classifySeverity.output.ticket_id       with severity = "low"       with reason = @orchestrator.crossPlatformTriage.output.summary       with description = @request.payload.message.parts[0].text   on_exit: ->     transition to @echo.unresolvedResponse echo unresolvedResponse:   kind: "a2a:response"   task: a2a.task({     state: "completed",     message: a2a.message({       messageId: uuid(),       parts: [         a2a.textPart("Ticket " + @generator.classifySeverity.output.ticket_id + " could not be resolved automatically and has been escalated to a human agent. Summary: " + @orchestrator.crossPlatformTriage.output.summary)       ]     }),     metadata: None   })` |
| :---- |

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAGsAAABMCAYAAABqFkmCAAAFLklEQVR4Xu2dT47VRhDG61IRx0gOAUcgN0guABeYHRtgFaREWaEgwSawyCazATSMIkCjgBAZkQgNqRHVLn+u6m57uv3v9U/6JPdX3eW2e/zc9vPz0EVjMxAajfXSBmtDVB+sm7/+dUFHz7N09PsZNm8oqg3WN3deDgZjjBpDig8W7vSraip/vjkf5NLiI35rEBpT+eHh68EOKaVv75/g6kzu/vH3oG2utgChMQXc8FryuP7T6aDuVK0ZQmMsuLG1hWC8hP759zOuZhUQGmPAjZxLc6z/1uO33YauBEIjl6vO9raiNUFo5HAoAyVaC4RGDrgxe1cKvkz4+fj95Wz0t+cfMFwMQiMFbsihSHh6+nEQy9HJu0/dTpwIoZECO9E0TnwKmQqhEYNnSLjypukaC6ERA1fWdHWNgdCIgStqKqPcazpCw2PMVx1N45VzLiM0PDB5U3mlblgTGh6YuKmOYl/AEhoemLSpnjwIDQ9M2FRXFoSGByZrqivrm2xCwwOTNdUXQmh4YKKmedQbg14pQrvOWkYa6pUSYKIt65fj9xdHT88G/trEd/nD/ldjkQQT1VDtdSEYX6NCX7tup7nKo165EtAvoXv/979m/loKfQ5LGdR8NlAkoF9CNXPXVOh3WEqACTxpnhnfqiJe3POZ7+6fDOLX4Ebo7Sfdd28cO357HmK8zPLaYm5dn7dJckic+6N59OJDrz33xYvlSKCwFAEbe/LIjfc6ZniaV+8+hThPFixkh3JdC47deHCK9iU8gLH1MxzjnW+R0zZXoU3X3AYbepK/OPS/V1P+s4//9WJ6Q8VLlS0Pyyx9ZOl6fN6Ssj4isJ72NHp7dEwPruhcPSyK9TmG9T3xAzmXOUJrA2wUEw8Eo3dGjnBdVhkHWY4kOXIEa4dhG+0JOIWXPzypL+iPPu3julJxz/eUHCxskCMEzy36Lw3BHLysZ28e1noZr28pz4p59Twf4x5Y31N0sPAvOVd4ombk40h/5PA5hAdCn/Qlhy7nDhZLn8QxhnljnhXz6nk+xj2wvqdQv2vagZWnSA8cl73c6OuynPytWWVM/LGm82Be9Hg92tcD77WN+bnxXIU8YekrU48q/NxnCbjs1ckpW7L6jO2wzPIukgWZTFh1WBz3YrrdmMmEpbCOsCSGUTlHMThufURqMI+UrY83AdsgeubGWH9Q3nlUTyYEbBtbdyrOEx7M5UmgrvlXw6icKwsd965J9IWioNt5AybxHx++xlDvaJMjCPsrkpmsMHXWp0nFuc+Yx5NAXfN57v0tIQH9LUg/I09hiQtG5a0LJzpbk4Z6BaPyVoXELpjXLA31CkblrUqj7yNuSQj1CkaDpmXE8weEegWjUdMysqBewWjUNL88SBfaE0zLK/ZrEhoYRoKmeZT6nRYNDCNJU32lBoohNPiKGRM11VXuL/kJDQaTNdWTfogzBaHB8C/wMGlTWaV+5WhBaAiYvKmMcs5NHoSGBlfUNF0lIDQ0bbIRF8PvasLrUy7LQy4lITQQ7gx2sil/BlcSQsOC32yJnT1kLTFQDKER49DeM2ip5ivqUhAaKVKv4N6zYvft5oDQGAOeWPesNUBolAY3emuyvgRcCkKjNGt5R6GQc3eG3xO/RgiNGuDOmFt7gdCoBe7AuVTj4nQpCI1alPwXFGO0JwiNmuCOrK29QWjUBndoLe0RQmMOcMeW1l4hNOaixg3itU65S0FozE2JF6EsfRtoLgiNpcAByNUhsZrB0sR+JxZ74e/eWeVgNWzaYG2IL0CWAdREOEfjAAAAAElFTkSuQmCC>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAACq0lEQVR4Xu2WyW4TQRCGwwkeI4cQ4n3fsphLOMBbwCGPiwARO/HYMXDIBa4RSAE69dXMtO2yxx4PcEH+pZJGvVT9tU313t4OO2yJj0/zweVhwV0+Q4ozkTX27Pm/BlGuhga5shvmK+6qUF0Q1oayNziCTN7Z+5kRHBw8xluUY+i6VHejcsONKk03qrZCqSBN3VMychay3LX6toK48QiP8E4Ni5Gg1nbjRtdNmj2R40h6uhbUOp7IUIgMjkoOHVZvKsAe43iExxieNHrupn3ipp0z96nXF3keSd9Nu2eyd6pkYiJXxZqSyBQJDbt4HhrvqKcY/vH+g0sCe5yZtI7duB6RIBKSDqt/LSg48kgo1fPIuMVIFI8Oi3ZZotFXEhDXdOTL2iXWTiIoIA29eEDYVxkHeImsAikhHRRpnAprZyXoZVot9p6c/76/t/oV6whwh5pg/1rSGEYhP7b2lkCo4txT2Uneg3UEgKZCIhjXAnVl7S2BQz78EsLP5y+tXo9NBL68eKX1E/g0pChGn3+5xOXvb99ZvR6bCHBXi7HaTl8HlsDPr9+sXo9NBLgbdkMWAhUI9Nzt6wur12MTgds3FxlSEBeh1IAWYTe5CIOi/J6FbBKWizAFAdpw9hPqaCtlxbQz34aVdG0I/AyIOiGpFdelQL1fCH+K/Mdgns9HgUJCoUUSAYbTTevEeJ/iHzAP8sVI1VqoxyRWR2IeEA2Nd2fDKE3xWYTjuKThC98BTETG8aka+XV3543yzRo5J+wQ9uM4l3Ecg/BBUpq9hCSfGo1G9BhpRcK3rLFHzgm7ep77gwdJjOn+/hPmOcMEj1CuERFD/OH4yfCtLyEMyxlyTti5a/VlBkVENJSIeIehBSmED1PObF1w20A6JGBg4SHGQomf5v/wWb7Df4sH9sUY3pmCteQAAAAASUVORK5CYII=>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,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>