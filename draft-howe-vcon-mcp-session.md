---
title: "vCon Extension for Model Context Protocol Sessions"
abbrev: "vCon MCP Session"
category: std
docname: draft-howe-vcon-mcp-session-latest
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

normative:
  RFC2119:
  RFC8174:
  RFC8259:
  I-D.draft-ietf-vcon-vcon-core:

informative:
  MCP:
    title: "Model Context Protocol Specification"
    author:
      - org: Anthropic
    target: https://modelcontextprotocol.io/
    date: 2024
  I-D.draft-rosenberg-ai-protocols:
  I-D.draft-howe-vcon-lifecycle:
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
- Does not require listing in the critical parameter
- Maintains backward compatibility with existing vCon implementations

## Extension Registration

This document defines the "mcp_session" extension token for registration in the vCon Extensions Names Registry:

- **Extension Name**: mcp_session
- **Extension Description**: Model Context Protocol session capture for human-AI conversations with tool invocations, resource access, and artifact production
- **Change Controller**: IESG
- **Specification Document**: This document

## Extension Usage

vCon instances that include MCP session data SHOULD include "mcp_session" in the extensions array:

~~~json
{
  "uuid": "01234567-89ab-cdef-0123-456789abcdef",
  "vcon": "0.0.1",
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
- **Specification Document**: This document, {{dialog-type-mcp_session}}

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

AI agents participating in MCP sessions SHOULD be represented as party objects with extended metadata:

~~~json
{
  "name": "Claude",
  "role": "assistant",
  "meta": {
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
  "role": "tool_provider",
  "meta": {
    "party_type": "mcp_server",
    "server_url": "https://mcp.notion.com/sse",
    "protocol_version": "2024-11-05",
    "capabilities": ["tools", "resources"],
    "tools_provided": ["notion_search", "notion_create_page", "notion_update_page"]
  }
}
~~~

## Party Type Values

The meta.party_type field SHOULD be one of:

- "human": Natural person (default if not specified)
- "ai_agent": AI assistant or agent
- "mcp_server": MCP protocol server
- "automated_system": Non-AI automated system

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
    "system_prompt_hash": "sha256-abc123...",
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
      "content_hash": "sha512-..."
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
    "content_hash": "sha512-..."
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

# Attachment Types for Tool Outputs

## Tool Output Attachments

Large tool outputs or produced artifacts SHOULD be stored as attachments:

~~~json
{
  "type": "mcp_artifact",
  "start": "2026-03-19T14:35:00Z",
  "party": 1,
  "dialog": 0,
  "mediatype": "application/pdf",
  "filename": "generated_report.pdf",
  "url": "https://storage.example.com/artifacts/report.pdf",
  "content_hash": "sha512-..."
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

## Data Subject Rights

MCP sessions may contain personal data subject to privacy regulations:

- Party identification enables data subject access requests
- Dialog content can be selectively redacted
- Tool outputs may need separate privacy assessment

## AI Agent Transparency

Users interacting with AI agents have a right to know:

- That they are interacting with an AI system
- What tools the AI has access to
- How their conversation data is stored

## Consent Management

MCP session capture SHOULD integrate with vCon consent management:

- Lawful basis attachments apply to MCP sessions
- Consent can be revoked, requiring coordinated deletion
- Session records support right to be forgotten requests

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
- **Specification Document**: This document, {{dialog-type-mcp_session}}

## Media Type Registration

This document requests registration of:

- **Type name**: application
- **Subtype name**: vnd.mcp-session+json
- **Required parameters**: None
- **Optional parameters**: version
- **Encoding considerations**: UTF-8
- **Security considerations**: See {{security-considerations}}
- **Interoperability considerations**: JSON format as defined in {{RFC8259}}
- **Published specification**: This document

# Examples

## Basic MCP Session vCon

~~~json
{
  "uuid": "0195a1b2-c3d4-e5f6-a7b8-c9d0e1f2a3b4",
  "vcon": "0.0.1",
  "extensions": ["mcp_session"],
  "created_at": "2026-03-19T14:30:00Z",
  "parties": [
    {
      "name": "Alice Smith",
      "tel": "+1-555-0100",
      "role": "user"
    },
    {
      "name": "Claude",
      "role": "assistant",
      "meta": {
        "party_type": "ai_agent",
        "provider": "anthropic",
        "model": "claude-4-opus"
      }
    },
    {
      "name": "Web Search",
      "role": "tool_provider",
      "meta": {
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

The authors would like to thank the vCon working group for their input on this specification.
