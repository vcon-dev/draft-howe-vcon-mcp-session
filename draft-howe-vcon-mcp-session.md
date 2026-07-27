---
title: "vCon Extension for Model Context Protocol Sessions"
abbrev: "vCon MCP Session"
category: std
docname: draft-howe-vcon-mcp-session-00
ipr: trust200902
area: "Applications and Real-Time"
workgroup: "vCon"
keyword: Internet-Draft
venue:
  group: "vCon"
  type: "Working Group"
  mail: "vcon@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/vcon/"
  github: "vcon-dev/draft-howe-vcon-mcp-session"
  latest: "https://vcon-dev.github.io/draft-howe-vcon-mcp-session/draft-howe-vcon-mcp-session-latest.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Thomas McCarthy-Howe
    organization: VCONIC
    email: ghostofbasho@gmail.com
 -
    name: Pavan Kumar
    organization: Independent
    email: pavanputhra@gmail.com

normative:
  RFC2119:
  RFC8174:
  RFC8259:
  I-D.draft-ietf-vcon-vcon-core:

informative:
  RFC6920:
  RFC6838:
  MCP:
    title: "Model Context Protocol Specification"
    author:
      - org: Anthropic
    target: https://modelcontextprotocol.io/
    date: 2024
  I-D.draft-howe-vcon-lawful-basis:

--- abstract

This document defines the Model Context Protocol (MCP) Session extension for Virtualized Conversations (vCon). The MCP Session extension provides a standardized framework for capturing human-AI conversations that involve tool invocations, resource access, and multi-turn reasoning within vCon containers. This extension enables consistent representation of AI agent interactions, preserving the complete context of tool usage, artifacts produced, and conversation flow while maintaining compatibility with existing vCon infrastructure for signing, encryption, and lifecycle management.

The MCP Session extension is designed as a Compatible Extension that introduces new dialog types and analysis formats for AI agent conversations without altering existing vCon semantics, ensuring backward compatibility with existing vCon implementations.

--- middle

# Requirements Language

{::boilerplate bcp14-tagged}

# Introduction

The Model Context Protocol (MCP) enables structured communication between AI agents and external services, allowing agents to access resources, execute tools, and perform complex multi-step operations on behalf of users. As AI agents become integral to business communications and customer interactions, there is a growing need to capture these interactions in a standardized, auditable format.

Virtual Conversations (vCon) provide a container format for conversational data that supports parties, dialog, analysis, and attachments. This document extends vCon to accommodate the unique characteristics of MCP sessions:

- **Multi-party interactions** involving humans, AI agents, and tool services
- **Structured message formats** with role-based attribution
- **Tool invocations** with inputs, outputs, and timing data
- **Resource access** patterns and retrieved content
- **Artifacts** produced during the conversation
- **Session context** including system prompts and configuration

## Use Cases

This extension supports the following use cases:

1. **Compliance and Audit**: Organizations using AI agents for customer service must maintain records of AI interactions for regulatory compliance.

2. **Quality Assurance**: Reviewing AI agent performance requires complete session records including tool usage patterns and decision points.

3. **Training Data Management**: AI training pipelines need structured conversation data with clear provenance and consent tracking.

4. **Dispute Resolution**: When AI agent actions are questioned, complete session records provide evidence of what occurred.

5. **Privacy Rights**: Data subject access requests require the ability to identify and export all AI interactions involving an individual.

# Conventions and Definitions

## Core Terms

**MCP Session**: A complete interaction between a user, an AI agent, and zero or more MCP servers, from initialization through completion or termination.

**AI Agent**: A software system utilizing a Large Language Model (LLM) to interact with users and perform tasks through reasoning and tool usage.

**MCP Server**: A service that provides resources, prompts, or tools to an AI agent via the Model Context Protocol.

**Tool Invocation**: A structured request from an AI agent to an MCP server to execute a specific function with given parameters.

**Resource**: Static content (files, database records, documents) accessed by an AI agent through an MCP server.

**Artifact**: Content produced by an AI agent or tool during a session, such as generated code, documents, or structured data.

**Session Context**: Configuration and state information that influences AI agent behavior, including system prompts, temperature settings, and available tools.

**Artifact**: A logical content object produced by an AI agent or tool during a session (such as generated code, a document, or structured data). An artifact MAY be conveyed inline within a message content block, or carried as a vCon Attachment when its size or independent lifecycle warrants it. The terms "artifact" and "attachment" are not interchangeable: the former is a logical concept defined here, the latter is a vCon container mechanism defined in {{I-D.draft-ietf-vcon-vcon-core}}.

## Protocol Version Strings

The `protocol_version` field carried in MCP session bodies and `meta.protocol_version` on MCP Server parties is the MCP protocol revision string as defined by the Model Context Protocol specification {{MCP}}. Because MCP is informatively referenced and revises independently of this document, vCon implementations MUST treat unknown `protocol_version` values as opaque, MUST NOT reject a dialog solely on that basis, and MUST preserve the value on round-trip.

## Hash Value Format

Hash values defined by this extension (for example `system_prompt_hash` and `content_hash`) use the form `"<algorithm>:<hex>"`, where `<algorithm>` is a token from the IANA Named Information Hash Algorithm Registry established by {{?RFC6920}} and `<hex>` is the lowercase hexadecimal encoding of the digest. Implementations MUST support `sha256` and MAY support additional registered algorithms. The digest is computed over the UTF-8 encoded byte sequence of the value being hashed (for `system_prompt_hash`, the system prompt text as delivered to the agent).

# MCP Session Overview

## Design Principles

The MCP Session extension follows these design principles:

**Completeness**: Capture sufficient detail to reconstruct the full interaction for audit and analysis purposes.

**Compatibility**: Work within existing vCon structures without requiring changes to core specifications.

**Extensibility**: Support provider-specific features through structured extension fields.

**Privacy-Aware**: Enable selective redaction of sensitive content while preserving structural integrity.

**Lifecycle Integration**: Support integration with vCon lifecycle management and SCITT transparency services.

## Hybrid Storage Approach

This extension uses a hybrid approach combining multiple vCon components:

1. **Dialog Objects** (type: "mcp_session"): Store the raw MCP session transcript
2. **Party Objects**: Represent human users, AI agents, and tool services
3. **Analysis Objects**: Store derived insights from MCP sessions
4. **Attachment Objects**: Store tool outputs and artifacts

This separation provides:
- Clear distinction between raw data and derived analysis
- Flexibility in what analysis is performed
- Support for compliance requirements needing raw session logs
- Efficient storage when multiple analyses reference the same session

# vCon MCP Session Extension Definition

## Extension Classification

The MCP Session extension is a *Compatible Extension* as defined in Section 2.5 of {{I-D.draft-ietf-vcon-vcon-core}}. This extension:

- Introduces new dialog types and analysis formats without altering existing vCon semantics
- Can be safely ignored by implementations that do not support MCP session processing
- MUST NOT be listed in the `critical` parameter; vcon-core requires only Incompatible Extensions to appear there
- Maintains backward compatibility with existing vCon implementations

## Extension Registration

This document defines the "mcp_session" extension token for registration in the vCon Extensions Names Registry:

- **Extension Name**: mcp_session
- **Extension Description**: Model Context Protocol session capture for human-AI conversations with tool invocations, resource access, and artifact production
- **Change Controller**: IESG
- **Specification Document**: This document

## Extension Usage

vCon instances that include MCP session data SHOULD include "mcp_session" in the extensions array. Note that {{I-D.draft-ietf-vcon-vcon-core}} deprecates the `vcon` schema-version parameter; the `extensions` array is the durable mechanism for negotiating MCP session support.

~~~json
{
  "uuid": "01234567-89ab-cdef-0123-456789abcdef",
  "vcon": "0.4.0",
  "extensions": ["mcp_session"],
  "created_at": "2026-03-19T14:30:00Z",
  "parties": [...],
  "dialog": [...],
  "analysis": [...],
  "attachments": [...]
}
~~~

# Dialog Type: mcp_session

## Type Registration

This document registers the dialog type "mcp_session" in the Dialog Object Types Registry:

- **Dialog Type Name**: mcp_session
- **Dialog Type Description**: Model Context Protocol session containing structured human-AI conversation with tool invocations
- **Change Controller**: IESG
- **Specification Document**: This document, Section "Dialog Type: mcp_session"

## Dialog Object Structure

An MCP session dialog object contains:

~~~json
{
  "type": "mcp_session",
  "start": "2026-03-19T14:30:00Z",
  "duration": 1847,
  "parties": [0, 1, 2],
  "originator": 0,
  "mediatype": "application/vnd.mcp-session+json",
  "body": { /* MCP Session Content */ },
  "encoding": "json"
}
~~~

## Required Parameters

**type**: MUST be "mcp_session"

**start**: ISO 8601 timestamp of session initialization

**parties**: Array of indices into the vCon parties array for all participants (human users, AI agents, tool services)

**mediatype**: MUST be "application/vnd.mcp-session+json"

**body** or **url**: Session content inline or externally referenced

**encoding**: MUST be "json" for inline content

## Optional Parameters

**duration**: Session duration in seconds

**originator**: Index of the party that initiated the session

**session_id**: Unique session identifier from the MCP client

**application**: Identifier for the MCP client application (e.g., "claude.ai", "cursor")

# Party Extensions for AI Agents

## AI Agent Party

The Party object schema in {{I-D.draft-ietf-vcon-vcon-core}} does not define a `role` parameter. To remain a Compatible Extension, this document carries MCP-specific role and provider information inside the per-party `meta` object rather than introducing new top-level Party parameters. Implementations that do not understand the MCP Session extension can safely ignore `meta` content per vcon-core.

AI agents participating in MCP sessions SHOULD be represented as party objects with extended metadata:

~~~json
{
  "name": "Claude",
  "meta": {
    "role": "assistant",
    "party_type": "ai_agent",
    "provider": "anthropic",
    "model": "claude-4-opus",
    "model_version": "2026-03",
    "capabilities": ["text", "vision", "tool_use", "computer_use"]
  }
}
~~~

## MCP Server Party

MCP servers providing tools or resources SHOULD be represented as:

~~~json
{
  "name": "Notion MCP Server",
  "meta": {
    "role": "mcp_server",
    "party_type": "mcp_server",
    "server_url": "https://mcp.notion.com/sse",
    "protocol_version": "2024-11-05",
    "capabilities": ["tools", "resources"],
    "tools_provided": ["notion_search", "notion_create_page", "notion_update_page"]
  }
}
~~~

The terms **MCP Server** and **mcp_server** are used consistently throughout this document for the party that provides tools or resources via MCP; the term "tool service" is not used.

## Party Type and Role Values

The `meta.party_type` field SHOULD be one of:

- "human": Natural person (default if not specified)
- "ai_agent": AI assistant or agent
- "mcp_server": MCP protocol server
- "automated_system": Non-AI automated system

The `meta.role` field SHOULD be one of "user", "assistant", "system", "tool", or "mcp_server". These values are scoped to this extension and are not registered as core Party Object Parameters. Implementations encountering an unknown `meta.party_type` or `meta.role` value MUST treat it as opaque, MUST NOT reject the vCon on that basis, and SHOULD preserve the value on round-trip.

# MCP Session Content Schema

## Top-Level Structure

The body of an mcp_session dialog MUST contain:

~~~json
{
  "session_id": "sess_abc123def456",
  "protocol_version": "2024-11-05",
  "client": {
    "name": "Claude Desktop",
    "version": "1.2.3"
  },
  "messages": [...]
}
~~~

## Required Fields

**session_id** (string): Unique identifier for the session

**protocol_version** (string): MCP protocol version used

**client** (object): Client application information
- name (string): Client name
- version (string): Client version

**messages** (array): Ordered array of message objects

## Optional Fields

**servers** (array): MCP servers involved in the session

~~~json
{
  "servers": [
    {
      "name": "filesystem",
      "url": "stdio://localhost",
      "capabilities": ["tools", "resources"],
      "tools": ["read_file", "write_file", "list_directory"]
    }
  ]
}
~~~

**context** (object): Session context and configuration

~~~json
{
  "context": {
    "system_prompt_hash": "sha256:abc123...",
    "temperature": 1.0,
    "max_tokens": 8192,
    "tools_available": ["web_search", "code_execution"]
  }
}
~~~

**artifacts** (array): Content produced during the session

~~~json
{
  "artifacts": [
    {
      "id": "artifact_001",
      "type": "code",
      "language": "python",
      "title": "data_analysis.py",
      "created_at": "2026-03-19T14:35:00Z",
      "content_hash": "sha512:..."
    }
  ]
}
~~~

## Message Object Structure

Each message in the messages array MUST contain:

~~~json
{
  "id": "msg_001",
  "role": "user",
  "timestamp": "2026-03-19T14:30:05Z",
  "content": [...]
}
~~~

**id** (string): Unique message identifier within the session

**role** (string): One of "user", "assistant", "system", "tool"

**timestamp** (string): ISO 8601 timestamp

**content** (array): Array of content blocks

## Content Block Types

The content block types defined below are the minimum set required by this extension. The MCP specification {{MCP}} may define additional content block types over time, and providers may carry vendor-specific blocks. Receivers MUST ignore unknown `content[].type` values without rejecting the dialog and MUST preserve them on round-trip.

### Text Content

~~~json
{
  "type": "text",
  "text": "Please analyze this data and create a visualization."
}
~~~

### Tool Use Content

~~~json
{
  "type": "tool_use",
  "id": "toolu_001",
  "name": "web_search",
  "input": {
    "query": "recent developments in quantum computing"
  }
}
~~~

### Tool Result Content

~~~json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "content": "...",
  "is_error": false,
  "duration_ms": 1234
}
~~~

### Image Content

~~~json
{
  "type": "image",
  "source": {
    "type": "base64",
    "media_type": "image/png",
    "data": "..."
  }
}
~~~

Or with external reference:

~~~json
{
  "type": "image",
  "source": {
    "type": "url",
    "url": "https://...",
    "content_hash": "sha512:..."
  }
}
~~~

# Analysis Types for MCP Sessions

## MCP Session Summary

Analysis providing a summary of the MCP session:

~~~json
{
  "type": "mcp_session_summary",
  "dialog": [0],
  "vendor": "anthropic",
  "schema": "mcp-summary-v1",
  "encoding": "json",
  "body": {
    "summary": "User requested data analysis...",
    "tools_used": ["web_search", "code_execution"],
    "tool_invocation_count": 5,
    "artifacts_produced": 2,
    "outcome": "completed",
    "duration_seconds": 1847
  }
}
~~~

## Tool Usage Analysis

Analysis of tool invocation patterns:

~~~json
{
  "type": "mcp_tool_analysis",
  "dialog": [0],
  "vendor": "anthropic",
  "schema": "mcp-tools-v1",
  "encoding": "json",
  "body": {
    "tools": [
      {
        "name": "web_search",
        "invocations": 3,
        "total_duration_ms": 4521,
        "error_count": 0
      }
    ]
  }
}
~~~

The schema identifiers `mcp-summary-v1` and `mcp-tools-v1` shown in the examples above are illustrative. Production deployments SHOULD set `schema` to a registered schema URI or a `vendor`-scoped name following the analysis schema guidance in {{I-D.draft-ietf-vcon-vcon-core}}, so that downstream consumers can validate analysis bodies against a stable contract.

# Attachment Types for Tool Outputs

## Tool Output Attachments

Tool outputs whose serialized JSON exceeds 16 KiB, and produced artifacts of any size, SHOULD be stored as vCon Attachments rather than inline in the dialog body. Storing large content as attachments keeps the dialog body indexable, avoids inflating the signed dialog scope, and lets attachments be revoked or redacted independently of the session transcript.

~~~json
{
  "type": "mcp_artifact",
  "start": "2026-03-19T14:35:00Z",
  "party": 1,
  "dialog": 0,
  "mediatype": "application/pdf",
  "filename": "generated_report.pdf",
  "url": "https://storage.example.com/artifacts/report.pdf",
  "content_hash": "sha512:..."
}
~~~

## Artifact Metadata

The attachment MAY include additional metadata in a meta object:

~~~json
{
  "type": "mcp_artifact",
  "mediatype": "text/x-python",
  "filename": "analysis.py",
  "body": "...",
  "encoding": "none",
  "meta": {
    "artifact_id": "artifact_001",
    "tool_use_id": "toolu_005",
    "artifact_type": "code",
    "language": "python"
  }
}
~~~

## Linking Attachments to Tool Invocations

When an attachment with `type: "mcp_artifact"` carries a `meta.tool_use_id`, that value MUST equal the `id` of a `tool_use` content block within an `mcp_session` dialog of the same vCon, and the attachment SHOULD set its `dialog` parameter to the index of that dialog. This linkage allows consumers to reconstruct the full provenance of an artifact (which message produced it, in which session, in response to which user input) without parsing the dialog body. Attachments without a `tool_use_id` represent artifacts not directly produced by a single tool invocation (for example, a final assistant-authored document) and SHOULD still set the `dialog` parameter when associated with a specific session.

# Security Considerations

## System Prompt Protection

System prompts often contain sensitive configuration. Implementations:

- SHOULD store only a hash of system prompts by default (system_prompt_hash)
- MAY include full system prompts when explicitly authorized
- MUST support redaction of system prompt content

## Tool Credential Handling

MCP sessions may involve tool invocations that use credentials:

- Tool inputs and outputs MUST NOT contain raw credentials
- Implementations SHOULD redact credential-like patterns
- OAuth tokens and API keys MUST be replaced with placeholders

## Content Integrity

MCP session content integrity is preserved through:

- vCon signing (JWS) for the complete container
- Content hashes for externally referenced content
- Session IDs for cross-reference validation

## Replay Prevention

Session records SHOULD include sufficient context to prevent misuse:

- Timestamps for all messages
- Session IDs bound to specific clients
- Tool invocation IDs for audit trails

# Privacy Considerations

MCP sessions are unusually privacy-sensitive among vCon dialog types because they capture not only what a human said, but what an AI agent inferred, retrieved, and acted upon on that human's behalf. A single record can therefore contain (a) personal data the user typed, (b) personal data about third parties that surfaced in tool results, (c) inferred attributes the user never disclosed, and (d) operational metadata such as system prompts and tool catalogs that can re-identify users by behavioral fingerprint even after direct identifiers are removed. This section gives normative guidance for handling each of these.

## Lawful Basis and Consent

MCP session capture SHOULD be governed by the same lawful-basis machinery vCon uses for other dialogs (see {{I-D.draft-howe-vcon-lawful-basis}}). Implementations MUST NOT rely on a single blanket consent for both the human-facing conversation and downstream tool invocations when those invocations transmit personal data to third-party MCP servers; each tool invocation that crosses an organizational or jurisdictional boundary SHOULD be supported by a documented basis. Where consent is the basis, revocation MUST propagate to derived analyses and artifacts, not only to the raw `mcp_session` dialog.

## Data Minimization

Implementations SHOULD capture the minimum MCP context required for the stated purpose. In particular:

- Tool result bodies SHOULD be omitted or truncated when the purpose of capture (for example, compliance attestation) does not require their content; the `tool_use_id` and a `content_hash` are usually sufficient for audit.
- `context.tools_available` SHOULD list tool names rather than full tool schemas when the schemas would expose sensitive endpoints, prompts, or business logic.
- System prompts SHOULD be referenced by `system_prompt_hash` rather than stored verbatim when the prompt itself is confidential. The hash preserves audit value without persisting prompt text.

## Data Subject Rights

The vCon `parties` array enables data subject access, rectification, and erasure requests to be answered. For MCP sessions specifically:

- Erasure of a human party MUST cascade to messages authored by that party, to tool invocations whose inputs were derived from that party's content, and to artifacts produced from those invocations.
- Rectification requests against tool outputs SHOULD be served by attaching a corrective analysis rather than mutating the original `mcp_session` dialog body, preserving evidentiary integrity.
- When erasure would compromise the integrity of a signed vCon, implementations SHOULD use the redaction mechanism described in {{I-D.draft-ietf-vcon-vcon-core}} rather than in-place modification.

## Third-Party Personal Data in Tool Results

Tool results frequently contain personal data about people who are not parties to the conversation (search results, CRM lookups, public records). Implementations:

- SHOULD treat such third parties as data subjects whose rights apply even though they are not represented in `parties`.
- SHOULD record the tool source and timestamp so that downstream deletion requests against the underlying source can be honored.
- MUST NOT use third-party data extracted from tool results as training data without an independent lawful basis.

## AI Agent Transparency

Where required by applicable law, the vCon SHOULD record evidence that the human party was informed they were interacting with an AI system, the categories of tools available to the agent, and the retention policy for the captured session. A `lawful_basis` attachment is the recommended carrier for this evidence.

## Re-identification and Behavioral Fingerprinting

Even with direct identifiers removed, MCP sessions are highly re-identifying: sequences of tool invocations, prompt phrasing, and timing form a behavioral fingerprint. Implementations that publish or share `mcp_session` data SHOULD assume re-identification risk is non-trivial and apply k-anonymity or differential-privacy techniques rather than relying on field-level redaction alone.

## Cross-Border Transfers

Tool invocations route data to MCP servers that may be operated in jurisdictions different from the human party's. Implementations SHOULD record, per tool invocation, sufficient information (for example, server identity in `meta.provider` and a region or jurisdiction hint) to support transfer-impact assessments after the fact.

# IANA Considerations

## vCon Extensions Names Registry

This document requests registration of:

- **Extension Name**: mcp_session
- **Extension Description**: Model Context Protocol session capture
- **Change Controller**: IESG
- **Specification Document**: This document

## Dialog Object Types Registry

This document requests registration of:

- **Dialog Type Name**: mcp_session
- **Dialog Type Description**: MCP session dialog
- **Change Controller**: IESG
- **Specification Document**: This document, Section "Dialog Type: mcp_session"

## Media Type Registration

This document requests registration of the following media type per {{?RFC6838}}:

- **Type name**: application
- **Subtype name**: vnd.mcp-session+json
- **Required parameters**: None
- **Optional parameters**: version (the MCP protocol revision string carried in the body's `protocol_version` field)
- **Encoding considerations**: binary; UTF-8 encoded JSON as defined in {{RFC8259}}
- **Security considerations**: See Section "Security Considerations" of this document.
- **Interoperability considerations**: JSON format as defined in {{RFC8259}}; receivers MUST ignore unknown content block types and unknown `meta` fields.
- **Published specification**: This document
- **Applications that use this media type**: vCon implementations capturing or processing MCP session content; AI agent platforms; conversational analytics, compliance, and audit tooling.
- **Fragment identifier considerations**: None defined.
- **Restrictions on usage**: None.
- **Provisional registration**: No.
- **Additional information**:
    - Deprecated alias names for this type: None
    - Magic number(s): None
    - File extension(s): .mcpsession.json
    - Macintosh file type code(s): None
- **Person & email address to contact for further information**: vCon working group <vcon@ietf.org>
- **Intended usage**: COMMON
- **Author/Change controller**: IETF
- **Author**: The authors of this document.

## Party Object Meta Field Guidance

This extension does not register new Party Object Parameter Names. The fields `meta.role`, `meta.party_type`, `meta.provider`, `meta.model`, `meta.model_version`, `meta.capabilities`, `meta.server_url`, `meta.protocol_version`, and `meta.tools_provided` are extension-defined and apply only when the vCon's `extensions` array includes `"mcp_session"`. Implementations that do not understand the MCP Session extension MUST ignore these fields per the meta-handling rules in {{I-D.draft-ietf-vcon-vcon-core}}.

# Examples

## Basic MCP Session vCon

~~~json
{
  "uuid": "0195a1b2-c3d4-e5f6-a7b8-c9d0e1f2a3b4",
  "vcon": "0.4.0",
  "extensions": ["mcp_session"],
  "created_at": "2026-03-19T14:30:00Z",
  "parties": [
    {
      "name": "Alice Smith",
      "tel": "+1-555-0100",
      "meta": {
        "role": "user"
      }
    },
    {
      "name": "Claude",
      "meta": {
        "role": "assistant",
        "party_type": "ai_agent",
        "provider": "anthropic",
        "model": "claude-4-opus"
      }
    },
    {
      "name": "Web Search",
      "meta": {
        "role": "mcp_server",
        "party_type": "mcp_server",
        "tools_provided": ["web_search"]
      }
    }
  ],
  "dialog": [
    {
      "type": "mcp_session",
      "start": "2026-03-19T14:30:00Z",
      "duration": 300,
      "parties": [0, 1, 2],
      "originator": 0,
      "mediatype": "application/vnd.mcp-session+json",
      "encoding": "json",
      "body": {
        "session_id": "sess_abc123",
        "protocol_version": "2024-11-05",
        "client": {
          "name": "Claude.ai",
          "version": "2026.03"
        },
        "messages": [
          {
            "id": "msg_001",
            "role": "user",
            "timestamp": "2026-03-19T14:30:05Z",
            "content": [
              {
                "type": "text",
                "text": "What are the latest developments in quantum computing?"
              }
            ]
          },
          {
            "id": "msg_002",
            "role": "assistant",
            "timestamp": "2026-03-19T14:30:10Z",
            "content": [
              {
                "type": "tool_use",
                "id": "toolu_001",
                "name": "web_search",
                "input": {
                  "query": "quantum computing breakthroughs 2026"
                }
              }
            ]
          },
          {
            "id": "msg_003",
            "role": "tool",
            "timestamp": "2026-03-19T14:30:15Z",
            "content": [
              {
                "type": "tool_result",
                "tool_use_id": "toolu_001",
                "content": "Recent developments include...",
                "is_error": false,
                "duration_ms": 1234
              }
            ]
          },
          {
            "id": "msg_004",
            "role": "assistant",
            "timestamp": "2026-03-19T14:30:20Z",
            "content": [
              {
                "type": "text",
                "text": "Based on my search, here are the latest developments..."
              }
            ]
          }
        ]
      }
    }
  ],
  "analysis": [
    {
      "type": "mcp_session_summary",
      "dialog": [0],
      "vendor": "anthropic",
      "encoding": "json",
      "body": {
        "summary": "User inquired about quantum computing developments. Assistant used web search to find current information.",
        "tools_used": ["web_search"],
        "tool_invocation_count": 1,
        "outcome": "completed"
      }
    }
  ]
}
~~~

--- back

# Acknowledgements

The authors thank Pavan Kumar for review and contributions, and the vCon working group for early input on this specification. Additional reviewers will be acknowledged in subsequent revisions.
