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

submissiontype: IETF
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
  RFC6973:
  I-D.draft-howe-vcon-lifecycle:
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

**Model**: The Large Language Model that produces completions on request. The Model is a distinct participant from the AI Agent: the Agent coordinates between the human, the Model, and MCP servers, while the Model reasons and chooses tools. Because each hop in a tool invocation has a different sender and recipient, this document represents the Model as its own party (see Section "Party Extensions for AI Agents").

**Tool Invocation**: A structured request to execute a specific function with given parameters. A complete invocation spans three hops: the Model requests a tool (`tool_use`), the Agent invokes it on an MCP server (`tool_call`), and the server returns a result (`tool_result`).

**Resource**: Static content (files, database records, documents) accessed by an AI agent through an MCP server.

**Roots**: The set of filesystem or context boundaries the MCP client advertises to servers for a session, each identified by a `file://` URI with an optional human-readable name.

**Session Context**: Configuration and state information that influences AI agent behavior, including system prompts, temperature settings, available tools, and the initial roots.

**Session ID**: The identifier the MCP client assigns to a session, carried as `session_id` in the dialog body and as the `session_id` dialog parameter. It is opaque to this specification: implementations MUST NOT parse it for structure and MUST preserve it on round-trip. Its purpose is to correlate a vCon dialog with the client-side session record it was captured from, and to correlate multiple vCons capturing the same session. A Session ID is unique within the scope of the issuing client; it is not globally unique and MUST NOT be relied upon as a security token (see Sections "Content Integrity" and "Replay Prevention").

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

1. **Dialog Objects** (type: "mcp_session"): Store the raw MCP session conversation
2. **Party Objects**: Represent human users, AI agents, models, and MCP servers
3. **Analysis Objects**: Store derived insights from MCP sessions
4. **Attachment Objects**: Carry the heavy payloads of tool outputs and artifacts referenced from the conversation

The term "conversation" is used rather than "transcript" throughout this document. "Transcript" is overloaded in the vCon ecosystem, where it commonly denotes the speech-to-text rendering of recorded audio; an MCP session conversation is natively textual and structured, not transcribed.

### Artifacts Live on the Timeline

An artifact is a message in the conversation, not a separate side table. Implementations MUST represent every artifact and every tool output as a message in the dialog body's `messages` array so that it appears in chronological order alongside the turns that produced it. When the payload is large (see Section "Tool Output Attachments"), the message carries a reference and the payload is spilled to a vCon Attachment. This yields a single rule: the timeline is authoritative for *what happened and when*, and attachments are a storage mechanism for *bulk*, never an alternative place for events to occur. Consumers reconstructing a session therefore need only walk `messages`, dereferencing attachments as needed, and never have to cross-reference a second collection to discover that something happened.

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
  "body": "{\"session_id\":\"sess_abc123def456\", ... }",
  "encoding": "json"
}
~~~

## Body Encoding and Example Conventions

{{I-D.draft-ietf-vcon-vcon-core}} defines `body` as a **string**, and `encoding` as a description of the encoding performed on that string value. The token `"json"` means the body string is itself JSON. Pairing `encoding: "json"` with a native JSON object is therefore invalid, however convenient it may look, and implementations MUST serialize MCP session content to a JSON string before assigning it to `body`.

Fully escaped JSON strings are unreadable at the length of a real session, so the examples in this document adopt a convention: where a body is shown as formatted JSON under the heading "decoded body content", that formatting is presentational only and the actual `body` value is the compact string serialization of the object shown. Examples that show a literal `body` value show it as a string. No example in this document is to be read as licensing an object-valued `body`.

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

The Party object schema in {{I-D.draft-ietf-vcon-vcon-core}} does not define a `role` parameter. To remain a Compatible Extension, this document carries MCP-specific participant and provider information inside the per-party `meta` object rather than introducing new top-level Party parameters. Implementations that do not understand the MCP Session extension can safely ignore `meta` content per vcon-core.

Earlier revisions of this document carried both `meta.role` and `meta.party_type` on the party object. The two axes largely restated one another and invited drift between them, so `meta.role` is no longer defined for parties: `meta.party_type` is the single normative axis for what kind of participant a party is. The `role` field on a *message* (Section "Message Object Structure") is unaffected and remains defined; it describes the conversational role of a turn, not the nature of a participant.

AI agents participating in MCP sessions SHOULD be represented as party objects with extended metadata:

~~~json
{
  "name": "Claude Desktop Agent",
  "meta": {
    "party_type": "ai_agent",
    "provider": "anthropic",
    "capabilities": ["text", "vision", "tool_use", "computer_use"]
  }
}
~~~

## Model Party

The Model is represented as a party distinct from the Agent. This is what makes the addressing model in Section "Message Object Structure" consistent: every message then has a real party on each end, and a `tool_use` emitted by the Model is distinguishable from the `tool_call` the Agent subsequently made.

~~~json
{
  "name": "claude-4-opus",
  "meta": {
    "party_type": "llm",
    "provider": "anthropic",
    "model": "claude-4-opus",
    "model_version": "2026-03"
  }
}
~~~

Implementations that cannot distinguish the Model from the Agent (for example when capturing from a client that does not expose the boundary) MAY represent both with a single `ai_agent` party and address Model-originated messages to and from it. Such a capture is lossy in exactly the way Section "Tool Call Content" describes: it cannot evidence that a requested tool was actually invoked.

## MCP Server Party

MCP servers providing tools or resources SHOULD be represented as:

~~~json
{
  "name": "Notion MCP Server",
  "meta": {
    "party_type": "mcp_server",
    "server_url": "https://mcp.notion.com/sse",
    "protocol_version": "2024-11-05",
    "capabilities": ["tools", "resources"],
    "tools_provided": [
      "notion_search",
      "notion_create_page",
      "notion_update_page"
    ]
  }
}
~~~

The terms **MCP Server** and **mcp_server** are used consistently throughout this document for the party that provides tools or resources via MCP; the term "tool service" is not used.

## Party Type Values

The `meta.party_type` field SHOULD be one of:

- "human": Natural person (default if not specified)
- "ai_agent": The coordinating agent, which mediates between the human, the Model, and MCP servers
- "llm": The Large Language Model that produces completions
- "mcp_server": MCP protocol server
- "automated_system": Non-AI automated system

These values are scoped to this extension and are not registered as core Party Object Parameters. Implementations encountering an unknown `meta.party_type` value MUST treat it as opaque, MUST NOT reject the vCon on that basis, and SHOULD preserve the value on round-trip.

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

**context** (object): Session context and configuration, including the boundary conditions the session opens with

~~~json
{
  "context": {
    "system_prompt_hash": "sha256:abc123...",
    "temperature": 1.0,
    "max_tokens": 8192,
    "tools_available": ["web_search", "code_execution"],
    "roots": [
      { "uri": "file:///home/user/projects/frontend",
        "name": "Frontend" },
      { "uri": "file:///home/user/projects/backend",
        "name": "Backend" }
    ]
  }
}
~~~

The `context.roots` array records the roots the client advertised at session initialization. Initial roots are session *context*, not an event: they establish the boundary the session opens with, and belong alongside the rest of the initial configuration. Roots that change mid-session are recorded as `root_updated` messages on the timeline instead (see Section "Root Updated Content"). Each root entry is a `file://` URI with an optional human-readable `name`, per {{MCP}}.

Note that MCP servers participating in the session are **not** listed in the dialog body. Servers are participants and are represented as parties in the vCon `parties` array (see Section "MCP Server Party"), referenced from messages by party index. Earlier revisions of this document also carried a `servers` array in the body, which duplicated information that belongs in `parties` and could disagree with it; that array is no longer defined.

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
  "party": 0,
  "to": 1,
  "role": "user",
  "timestamp": "2026-03-19T14:30:05Z",
  "content": [...]
}
~~~

**id** (string): Unique message identifier within the session

**party** (integer): Index into the vCon `parties` array identifying the party that emitted this message. This parameter is REQUIRED. Without it a consumer cannot reconstruct who emitted a message, which defeats the audit purpose of the extension. The value MUST be a valid index into `parties`, and that party SHOULD also appear in the dialog object's `parties` array.

**to** (integer or array of integers): Index or indices into the vCon `parties` array identifying the party the message is addressed to. Messages in an MCP session are point-to-point far more often than they are broadcast: a `tool_call` is addressed to one specific MCP server, and the Model's final answer is addressed to the human. Consumers use `to` to decide what to surface, since tool-negotiation traffic addressed to an MCP server is typically optional to display while a response addressed to the human is not. This parameter SHOULD be present; it MAY be omitted only for a genuinely broadcast message, in which case the dialog object's `parties` array is the implied audience.

**role** (string): One of "user", "assistant", "system", "tool". The `role` describes the conversational role of the turn and is retained for alignment with MCP and with LLM provider APIs. Where `role` and the emitting party's `meta.party_type` disagree, `party` is authoritative for provenance.

**timestamp** (string): ISO 8601 timestamp

**content** (array): Array of content blocks

Every message therefore names a real party on each end. This is what allows a consumer to answer "who did this, and to whom" for each hop of a tool invocation without inference.

## Content Block Types

The content block types defined below are the minimum set required by this extension. The MCP specification {{MCP}} may define additional content block types over time, and providers may carry vendor-specific blocks. Receivers MUST ignore unknown `content[].type` values without rejecting the dialog and MUST preserve them on round-trip.

The `party` and `to` indices shown in the examples in this section are illustrative and local to each example; they are not the indices used in the worked example in Section "Examples". In these snippets, party 0 is the Agent and party 2 is an MCP server.

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

### Tool Call Content

A `tool_use` block records that the Model *asked* for a tool. It does not record that the tool was actually invoked. The `tool_call` block records the Agent actually invoking the tool on an MCP server:

~~~json
{
  "type": "tool_call",
  "id": "toolu_001",
  "name": "web_search",
  "input": {
    "query": "recent developments in quantum computing"
  }
}
~~~

A complete tool invocation therefore spans three hops, each with a different sender and a different recipient:

1. `tool_use` — the Model chooses a tool and emits the request (Model to Agent)
2. `tool_call` — the Agent invokes the tool on the MCP server (Agent to MCP server)
3. `tool_result` — the MCP server returns the result (MCP server to Agent)

Collapsing these into two blocks loses who did what. Specifically, it makes the case where the Model requested a call that the Agent never actually made indistinguishable from the case where no call was requested at all. That distinction is load-bearing for audit: it is the difference between "the agent did something on the user's behalf" and "the model proposed something that was declined, filtered, or dropped". Implementations MUST emit a `tool_call` for each invocation actually dispatched to an MCP server. A `tool_use` with no corresponding `tool_call` sharing its `id` is a valid and meaningful record, and consumers MUST NOT treat it as an error.

The `id` of a `tool_call` MUST equal the `id` of the `tool_use` block it discharges, so that the three hops of one invocation can be correlated.

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

The `tool_use_id` correlates the result with the `tool_use` and `tool_call` blocks bearing that `id`. Where the result payload is large, `content` is replaced by a reference to a vCon Attachment (see Section "Linking Attachments to Tool Invocations").

### File Content

A single `file` content block carries any attached media. Earlier revisions of this document defined a dedicated `image` block. That does not scale: adding `audio` and `video` would produce three near-identical block types differing only in MIME prefix, and every subsequent format (PDF, slide deck, or anything not yet invented) would demand another. The `media_type` field already tells a receiver what kind of file is attached, so one block type suffices and the schema does not grow as formats arrive.

~~~json
{
  "type": "file",
  "source": {
    "type": "base64",
    "media_type": "image/png",
    "data": "..."
  }
}
~~~

Or with external reference, which SHOULD point to a vCon Attachment for large media:

~~~json
{
  "type": "file",
  "source": {
    "type": "url",
    "url": "https://...",
    "media_type": "audio/wav",
    "filename": "voice_clip.wav",
    "content_hash": "sha512:..."
  }
}
~~~

A conversational turn frequently mixes text with attached media: a human uploads a PDF and asks for a summary, sends a voice clip with a follow-up question, or the Model replies with generated audio. Such a turn carries both a `text` block and one or more `file` blocks in its `content` array. Implementations that previously emitted `"type": "image"` SHOULD emit `"type": "file"` with the appropriate `media_type`; receivers SHOULD accept `"image"` as a deprecated alias and preserve it on round-trip.

### Prompt Content

MCP servers can expose prompts. Retrieval of a prompt is recorded as a request and result pair mirroring MCP `prompts/get`. The request carries `name` and `arguments`; the result carries `description` and a `messages` array in which each message's `content` is a typed block, not a bare string:

~~~json
[
  { "type": "prompt_request", "party": 0, "to": 2,
    "content": { "name": "summarize_call",
                 "arguments": { "style": "brief" } } },
  { "type": "prompt_result", "party": 2, "to": 0,
    "content": { "description": "Call summary prompt",
                 "messages": [
                   { "role": "user",
                     "content": { "type": "text",
                                  "text": "Summarize this call: ..."
                                } }
                 ] } }
]
~~~

### Resource Content

MCP servers can expose resources. Retrieval is recorded as a request and result pair mirroring MCP `resources/read`. The request carries a `uri`; the result carries a `contents` array in which each entry has `uri` and `mimeType` and either `text` for textual content or `blob` for base64 binary content. For large binaries the `blob` SHOULD be spilled to a vCon Attachment and referenced rather than inlined:

~~~json
[
  { "type": "resource_request", "party": 0, "to": 2,
    "content": { "uri": "file:///recordings/abc123.wav" } },
  { "type": "resource_result", "party": 2, "to": 0,
    "content": { "contents": [
                   { "uri": "file:///recordings/abc123.wav",
                     "mimeType": "audio/wav",
                     "blob": "..." }
                 ] } }
]
~~~

### Root Updated Content

Roots can change during a session. The initial roots belong in `context.roots` (see Section "Optional Fields"); a change is an event and is recorded on the timeline so that the boundary in effect at any point in the session can be reconstructed:

~~~json
{ "type": "root_updated", "party": 0, "to": 2,
  "content": { "roots": [
    { "uri": "file:///home/user/projects/frontend",
      "name": "Frontend" },
    { "uri": "file:///home/user/projects/backend",
      "name": "Backend" }
  ] } }
~~~

This maps to MCP, in which the client advertises roots and emits `notifications/roots/list_changed` when they change, after which the server re-reads them via `roots/list`. The `content.roots` array carries the complete new set, not a delta.

### Sampling Content

An MCP server can ask the client to run a model completion. Sampling is **server-initiated**: the request is addressed from the MCP server to the Agent, the reverse of the tool-invocation direction. The request mirrors MCP `sampling/createMessage`, carrying `messages` of typed content blocks, `modelPreferences`, `systemPrompt`, and `maxTokens`; the response carries `role`, a typed `content` block, `model`, and `stopReason`. Note that MCP `stopReason` values are camelCase (for example `endTurn`, not `end_turn`):

~~~json
[
  { "type": "sampling_request", "party": 2, "to": 0,
    "content": { "messages": [
                   { "role": "user",
                     "content": { "type": "text",
                                  "text": "Draft a reply" } }
                 ],
                 "modelPreferences": {
                   "hints": [ { "name": "claude-3-sonnet" } ],
                   "intelligencePriority": 0.8 },
                 "systemPrompt": "You are a helpful assistant.",
                 "maxTokens": 512 } },
  { "type": "sampling_response", "party": 0, "to": 2,
    "content": { "role": "assistant",
                 "content": { "type": "text",
                              "text": "Here's a draft ..." },
                 "model": "claude-3-sonnet-20240307",
                 "stopReason": "endTurn" } }
]
~~~

Sampling is privacy-relevant in a way ordinary tool traffic is not: it lets a third-party MCP server put text of its choosing in front of the Model on the human's behalf. Implementations SHOULD record sampling exchanges in full where a lawful basis permits, and MUST NOT omit them merely because they were server-initiated.

# Analysis Types for MCP Sessions

## MCP Session Summary

Analysis providing a summary of the MCP session. Note that `body` is a JSON string, per Section "Body Encoding and Example Conventions":

~~~json
{
  "type": "mcp_session_summary",
  "dialog": [0],
  "vendor": "anthropic",
  "schema": "mcp-summary-v1",
  "encoding": "json",
  "body": "{\"summary\":\"User requested ... \", ... }"
}
~~~

The decoded body content of the analysis above is:

~~~json
{
  "summary": "User requested data analysis...",
  "tools_used": ["web_search", "code_execution"],
  "tool_invocation_count": 5,
  "artifacts_produced": 2,
  "outcome": "completed",
  "duration_seconds": 1847
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
  "body": "{\"tools\":[{\"name\":\"web_search\", ... }]}"
}
~~~

The decoded body content is:

~~~json
{
  "tools": [
    {
      "name": "web_search",
      "invocations": 3,
      "total_duration_ms": 4521,
      "error_count": 0
    }
  ]
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
    "tool_result_id": "toolu_005",
    "message_index": 7,
    "artifact_type": "code",
    "language": "python"
  }
}
~~~

## Linking Attachments to Tool Invocations

An attachment carrying a tool output MUST be linked to the timeline in both directions.

**Forward, timeline to attachment.** A placeholder message MUST remain on the timeline where the output occurred, carrying a reference to the attachment rather than the payload. This preserves the rule in Section "Artifacts Live on the Timeline": the event stays in chronological order even when its bulk does not.

**Reverse, attachment to timeline.** The attachment MUST carry a `meta.message_index` giving the index within the dialog body's `messages` array of the message the output belongs to.

~~~json
{ "type": "tool_result", "party": 2, "to": 0,
  "content": { "id": "call_42", "ref": "attachment:0" } }
~~~

~~~json
{
  "type": "mcp_artifact",
  "dialog": 0,
  "mediatype": "application/json",
  "encoding": "json",
  "body": "{ ...large tool result... }",
  "meta": {
    "message_index": 5,
    "tool_result_id": "call_42"
  }
}
~~~

When an attachment carries a `meta.tool_result_id`, that value MUST equal the `tool_use_id` of a `tool_result` content block within an `mcp_session` dialog of the same vCon, and the attachment SHOULD set its `dialog` parameter to the index of that dialog.

Earlier revisions of this document keyed this linkage off the `tool_use` block instead. That was the wrong end of the invocation: the attachment carries the tool's *output*, which is produced by the `tool_result`, not requested by the `tool_use`. Keying off the request made reconstruction needlessly expensive, since a consumer starting from a `tool_use` had to scan the message array for a matching `tool_result` and then fall back to the attachments if the result was not inline. Keying off the result, with a placeholder `tool_result` always present on the timeline, makes provenance a direct lookup in both directions.

Attachments not produced by a single tool invocation (for example a final assistant-authored document) carry no `tool_result_id`, and SHOULD still set both `dialog` and `meta.message_index` for the message that produced them.

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
- The Session ID (see Section "Core Terms"), which lets a consumer confirm that a dialog, its analyses, and its attachments all refer to the same captured session

The Session ID contributes correlation, not integrity. It is client-assigned and unauthenticated, so it MUST NOT be treated as evidence of provenance on its own; only the container signature provides that.

## Replay Prevention

Session records SHOULD include sufficient context to prevent misuse:

- Timestamps for all messages
- The Session ID, scoped to the issuing client (see Section "Core Terms"), so that a record replayed under a different client's identity can be detected
- Tool invocation IDs for audit trails

Because a Session ID is unique only within the scope of its issuing client, implementations MUST NOT use it as a bearer token or as the sole key for authorization decisions, and MUST NOT assume two vCons bearing the same Session ID originated from the same client unless the container signatures agree.

# Privacy Considerations

MCP sessions are unusually privacy-sensitive among vCon dialog types because they capture not only what a human said, but what an AI agent inferred, retrieved, and acted upon on that human's behalf. A single record can therefore contain (a) personal data the user typed, (b) personal data about third parties that surfaced in tool results, (c) inferred attributes the user never disclosed, and (d) operational metadata such as system prompts and tool catalogs that can re-identify users by behavioral fingerprint even after direct identifiers are removed. This section gives normative guidance for handling each of these.

## Lawful Basis and Consent

MCP session capture SHOULD be governed by the same lawful-basis machinery vCon uses for other dialogs (see {{I-D.draft-howe-vcon-lawful-basis}}). Implementations MUST NOT rely on a single blanket consent for both the human-facing conversation and downstream tool invocations when those invocations transmit personal data to third-party MCP servers; each tool invocation that crosses an organizational or jurisdictional boundary SHOULD be supported by a documented basis.

Where consent is the basis, revocation MUST propagate to derived analyses and artifacts, not only to the raw `mcp_session` dialog. This obligation requires a mechanism, and the lawful-basis document does not supply one: it records the basis and models per-purpose grants, and lists withdrawal of a basis as a requirement, but does not define how a withdrawal propagates to derived objects. Implementations satisfy the requirement above as follows.

*Within a vCon*, the derivation links this document already mandates are the propagation path. An analysis names the dialogs it derives from in its `dialog` parameter; a tool-output or artifact attachment names its originating message via `meta.message_index` and, where applicable, `meta.tool_result_id` (see Section "Linking Attachments to Tool Invocations"). On revocation, an implementation MUST walk these links from the revoked content and apply the same disposition to every object reachable through them. Because the links are required rather than advisory, this walk is well-defined without further specification.

*Across parties*, propagation is out of scope for a single container and MUST use the coordinated deletion mechanism in {{I-D.draft-howe-vcon-lifecycle}}, which defines how a revocation reaches parties that already received a copy of the vCon. Implementations that capture MCP sessions and distribute them to third parties SHOULD therefore support the lifecycle extension; without it, a revocation cannot be honored beyond the local container and the requirement in this section cannot be met in full.

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

A right that cannot be located cannot be exercised. Treating third parties as data subjects therefore requires a defined, searchable place to record the reference, which earlier revisions of this document omitted. Implementations that identify a third-party data subject in a tool result SHOULD record it as an analysis object of type `mcp_third_party_subjects` against the dialog, with `encoding: "json"` and a body whose decoded content is an array of subject records:

~~~json
{
  "subjects": [
    {
      "message_index": 5,
      "identifier": "sha256:...",
      "identifier_type": "email",
      "source": "crm_lookup",
      "source_party": 2,
      "observed_at": "2026-03-19T14:30:15Z"
    }
  ]
}
~~~

The `identifier` SHOULD be a salted hash rather than the plaintext identifier, so that a subject-access or erasure request can be matched by recomputing the hash without the index itself becoming an additional store of third-party personal data. `message_index` locates the message the reference appeared in, making erasure actionable through the same derivation walk described in Section "Lawful Basis and Consent". Recording subjects in a single, typed analysis object rather than scattering them through tool bodies is what makes them indexable across a corpus, which is the practical precondition for honoring a request at all.

## AI Agent Transparency

Where required by applicable law, the vCon SHOULD record evidence that the human party was informed they were interacting with an AI system, the categories of tools available to the agent, and the retention policy for the captured session. A `lawful_basis` attachment is the recommended carrier for this evidence.

## Re-identification and Behavioral Fingerprinting

Even with direct identifiers removed, MCP sessions are highly re-identifying: sequences of tool invocations, prompt phrasing, and timing form a behavioral fingerprint. This is an instance of the correlation and identification threats described in {{?RFC6973}}, and implementations that publish or share `mcp_session` data SHOULD assume re-identification risk is non-trivial rather than assuming field-level redaction is sufficient.

This document deliberately does not mandate a specific de-identification technique. Earlier revisions directed implementers to "apply k-anonymity or differential-privacy techniques" without citing a reference, which is not actionable: there is no IETF specification defining how to apply either method to structured conversational data, the two are not interchangeable, and neither is known to be sufficient against fingerprinting that arises from tool-invocation *sequence* rather than from field values. An implementer unfamiliar with these methods could not determine from that text what to actually do, and an implementer familiar with them would be right to doubt that either one closes the gap.

What this document can require concretely: implementations MUST NOT treat removal of direct identifiers as rendering an `mcp_session` non-personal; they SHOULD perform and record a re-identification risk assessment before publishing or sharing session data beyond the original parties; and they SHOULD reduce fingerprint surface by the minimization measures in Section "Data Minimization", which act on the sequence and schema exposure that drive the risk, rather than only on field values.

Selecting an appropriate de-identification technique for conversational and tool-invocation data is an open question that warrants input from privacy specialists, and the authors solicit it. Future revisions should either cite a concrete published method with stated assumptions and limitations, or state plainly that no adequate general method is known for this data shape.

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

This extension does not register new Party Object Parameter Names. The fields `meta.party_type`, `meta.provider`, `meta.model`, `meta.model_version`, `meta.capabilities`, `meta.server_url`, `meta.protocol_version`, and `meta.tools_provided` are extension-defined and apply only when the vCon's `extensions` array includes `"mcp_session"`. Implementations that do not understand the MCP Session extension MUST ignore these fields per the meta-handling rules in {{I-D.draft-ietf-vcon-vcon-core}}.

# Examples

## Basic MCP Session vCon

This example shows four parties, all three hops of a tool invocation, a human turn that mixes text with an attached file, and a large tool result spilled to an attachment with links in both directions.

Per Section "Body Encoding and Example Conventions", the `body` values below are JSON strings; the session content is shown decoded immediately afterward for readability.

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
        "party_type": "human"
      }
    },
    {
      "name": "Claude Desktop Agent",
      "meta": {
        "party_type": "ai_agent",
        "provider": "anthropic"
      }
    },
    {
      "name": "claude-4-opus",
      "meta": {
        "party_type": "llm",
        "provider": "anthropic",
        "model": "claude-4-opus"
      }
    },
    {
      "name": "Web Search",
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
      "parties": [0, 1, 2, 3],
      "originator": 0,
      "mediatype": "application/vnd.mcp-session+json",
      "encoding": "json",
      "body": "{\"session_id\":\"sess_abc123\", ... }"
    }
  ],
  "analysis": [
    {
      "type": "mcp_session_summary",
      "dialog": [0],
      "vendor": "anthropic",
      "schema": "mcp-summary-v1",
      "encoding": "json",
      "body": "{\"summary\":\"User asked about quantum ... \", ... }"
    }
  ],
  "attachments": [
    {
      "type": "mcp_artifact",
      "dialog": 0,
      "mediatype": "application/json",
      "encoding": "json",
      "body": "{\"results\":[ ...large search result... ]}",
      "meta": {
        "message_index": 3,
        "tool_result_id": "toolu_001"
      }
    }
  ]
}
~~~

The decoded body content of the dialog object is:

~~~json
{
  "session_id": "sess_abc123",
  "protocol_version": "2024-11-05",
  "client": {
    "name": "Claude.ai",
    "version": "2026.03"
  },
  "context": {
    "system_prompt_hash": "sha256:abc123...",
    "tools_available": ["web_search"]
  },
  "messages": [
    {
      "id": "msg_001",
      "party": 0,
      "to": 1,
      "role": "user",
      "timestamp": "2026-03-19T14:30:05Z",
      "content": [
        {
          "type": "text",
          "text": "Quantum computing news? Compare to the paper."
        },
        {
          "type": "file",
          "source": {
            "type": "url",
            "url": "https://storage.example.com/uploads/paper.pdf",
            "media_type": "application/pdf",
            "filename": "survey_2025.pdf",
            "content_hash": "sha512:..."
          }
        }
      ]
    },
    {
      "id": "msg_002",
      "party": 2,
      "to": 1,
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
      "party": 1,
      "to": 3,
      "role": "assistant",
      "timestamp": "2026-03-19T14:30:11Z",
      "content": [
        {
          "type": "tool_call",
          "id": "toolu_001",
          "name": "web_search",
          "input": {
            "query": "quantum computing breakthroughs 2026"
          }
        }
      ]
    },
    {
      "id": "msg_004",
      "party": 3,
      "to": 1,
      "role": "tool",
      "timestamp": "2026-03-19T14:30:15Z",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_001",
          "content": { "ref": "attachment:0" },
          "is_error": false,
          "duration_ms": 1234
        }
      ]
    },
    {
      "id": "msg_005",
      "party": 2,
      "to": 0,
      "role": "assistant",
      "timestamp": "2026-03-19T14:30:20Z",
      "content": [
        {
          "type": "text",
          "text": "Based on my search, here are the developments..."
        }
      ]
    }
  ]
}
~~~

Reading the example: `msg_002` is the Model asking party 1 for a search, and `msg_003` is the Agent actually issuing it to party 3. Had the Agent declined or dropped the request, `msg_002` would appear with no `msg_003` following it, and that fact would be evident from the record rather than inferred from its absence. The `tool_result` in `msg_004` is a placeholder carrying a reference; the payload sits in attachment 0, which points back with `meta.message_index: 3`, the index of `msg_004` in the `messages` array.

--- back

# Acknowledgements

The authors thank the vCon working group for early input on this specification. The message addressing model, the three-hop tool invocation sequence, the generalization of the image content block to a file block, and the correction of the attachment linkage to key off the tool result all originate in a detailed review by Pavan Kumar. Additional reviewers will be acknowledged in subsequent revisions.
