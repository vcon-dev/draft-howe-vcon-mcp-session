# Integrating MCP Sessions into VCONs: Analysis and Options

## Executive Summary

This document analyzes approaches for incorporating Model Context Protocol (MCP) sessions as content within Virtual Conversation (vCon) containers. MCP sessions represent a new form of structured human-AI dialog that fits naturally within the vCon framework's existing architecture for capturing conversations, while also introducing unique characteristics that require careful consideration.

---

## 1. Background

### 1.1 What is a VCON?

A vCon is a standardized JSON container for conversational data with four primary components:

- **Parties**: Participant identity information (names, identifiers, contact info)
- **Dialog**: The actual conversation content (text, audio, video recordings)
- **Analysis**: Derived data (transcripts, summaries, sentiment analysis)
- **Attachments**: Ancillary documents and supporting files

VCONs support inline content (base64 encoded) or external references (URLs with content hashes), can be signed (JWS) or encrypted (JWE), and have an extension mechanism for adding new capabilities.

### 1.2 What is MCP?

The Model Context Protocol enables operations between a client (typically an AI agent) and servers providing:

- **Resources**: Files, database records, static content
- **Prompts**: AI prompts that perform transactional operations
- **Tools**: Programmatic APIs with defined inputs/outputs

MCP sessions capture the structured interaction between users, AI agents, and external services, including tool invocations, resource access, and multi-turn reasoning.

### 1.3 Key Differences from Traditional Conversations

| Aspect | Traditional Call/Chat | MCP Session |
|--------|----------------------|-------------|
| Participants | Humans only | Human + AI agent(s) + tools |
| Content | Linear text/audio | Structured messages, tool calls, results |
| Metadata | Time, duration, parties | Context, tool definitions, state |
| Flow | Sequential turns | Branching, async tool execution |
| Artifacts | None or files shared | Code, structured data, tool outputs |

---

## 2. Integration Options

### Option A: New Dialog Type "mcp_session"

**Approach**: Register a new dialog type alongside "recording", "text", "transfer", and "incomplete".

```json
{
  "dialog": [
    {
      "type": "mcp_session",
      "start": "2026-01-01T14:30:00Z",
      "duration": 1847,
      "parties": [0, 1],
      "mediatype": "application/vnd.mcp+json",
      "body": { /* MCP session content */ },
      "encoding": "json"
    }
  ]
}
```

**Pros**:
- Treats MCP session as primary conversation content (which it is)
- Follows established vCon patterns
- Dialog reference indexes work naturally
- Existing analysis objects can reference the dialog

**Cons**:
- Requires IANA registration of new dialog type
- May need additional parameters beyond current dialog spec
- "dialog" semantically implies human-to-human exchange

**Recommended for**: When the MCP session IS the primary conversation being captured.


### Option B: Attachment with Custom Type

**Approach**: Store MCP session data as an attachment with type "mcp_session".

```json
{
  "attachments": [
    {
      "type": "mcp_session",
      "start": "2026-01-01T14:30:00Z",
      "party": 0,
      "dialog": 0,
      "mediatype": "application/vnd.mcp+json",
      "body": { /* MCP session content */ },
      "encoding": "json"
    }
  ]
}
```

**Pros**:
- No changes to core vCon schema
- Attachments are designed for "opaque objects"
- Can link to specific dialogs via the dialog parameter
- Flexible, implementation can evolve

**Cons**:
- Attachments are for "ancillary documents" not primary content
- Less semantic precision about what the MCP session represents
- Analysis objects cannot directly reference attachments

**Recommended for**: When MCP session is supplementary to another primary conversation (e.g., AI-assisted call where the human call is primary).


### Option C: Compatible Extension "mcp_session"

**Approach**: Define a formal vCon extension following the extension framework.

```json
{
  "uuid": "...",
  "extensions": ["mcp_session"],
  "created_at": "2026-01-01T14:30:00Z",
  "parties": [...],
  "dialog": [...],
  "mcp_sessions": [
    {
      "session_id": "sess_abc123",
      "start": "2026-01-01T14:30:00Z",
      "duration": 1847,
      "parties": [0, 1],
      "context": { /* session context */ },
      "messages": [ /* structured message array */ ],
      "tools_used": [ /* tool definitions and invocations */ ],
      "artifacts": [ /* produced outputs */ ]
    }
  ]
}
```

**Pros**:
- Full control over schema design
- Can define MCP-specific parameters optimally
- Compatible extension pattern (safe to ignore)
- Follows existing extension examples (lawful_basis, wtf_transcription)
- Can be combined with analysis objects for derived data

**Cons**:
- Requires extension specification document
- IANA registration needed
- Adds complexity to vCon structure

**Recommended for**: Comprehensive MCP session capture with full fidelity.


### Option D: Analysis Object for Processed MCP Data

**Approach**: Store structured analysis derived from MCP sessions.

```json
{
  "analysis": [
    {
      "type": "mcp_session_analysis",
      "dialog": [0],
      "vendor": "anthropic",
      "schema": "mcp-analysis-v1",
      "mediatype": "application/json",
      "body": {
        "tools_invoked": [...],
        "decisions_made": [...],
        "outcomes": [...],
        "confidence_scores": [...]
      },
      "encoding": "json"
    }
  ]
}
```

**Pros**:
- Analysis is appropriate for derived/processed data
- Links to source dialog(s)
- Vendor and schema fields support provider variation
- Existing analysis patterns apply

**Cons**:
- Analysis is for "derived" data, not raw conversation
- Raw MCP session would need separate storage
- Less suitable for audit/compliance needs

**Recommended for**: When you need processed/summarized MCP interaction data, not raw session logs.


### Option E: Hybrid Approach (Recommended)

**Approach**: Combine dialog type for raw session + analysis for derived insights.

```json
{
  "uuid": "...",
  "extensions": ["mcp_session"],
  "parties": [
    { "name": "Alice Smith", "role": "user" },
    { "name": "Claude", "role": "assistant", "meta": { "model": "claude-4" } }
  ],
  "dialog": [
    {
      "type": "mcp_session",
      "start": "2026-01-01T14:30:00Z",
      "duration": 1847,
      "parties": [0, 1],
      "mediatype": "application/vnd.mcp+json",
      "body": { /* Raw MCP session */ },
      "encoding": "json"
    }
  ],
  "analysis": [
    {
      "type": "mcp_summary",
      "dialog": [0],
      "vendor": "anthropic",
      "body": { /* Processed insights */ },
      "encoding": "json"
    }
  ],
  "attachments": [
    {
      "type": "mcp_tool_output",
      "dialog": 0,
      "filename": "generated_report.pdf",
      "url": "https://...",
      "content_hash": "sha512-..."
    }
  ]
}
```

**Pros**:
- Clear separation of concerns
- Raw session preserved for compliance
- Analysis enables processing pipeline
- Attachments capture tool outputs
- Extensible for future needs

**Cons**:
- More complex implementation
- Requires coordinated extension spec

---

## 3. MCP Session Content Schema

Regardless of storage approach, the MCP session content should capture:

### 3.1 Required Fields

```json
{
  "session_id": "string",           // Unique session identifier
  "protocol_version": "string",     // MCP protocol version
  "start_time": "ISO8601",          // Session start
  "end_time": "ISO8601",            // Session end (if completed)
  "client": {                       // Client information
    "name": "string",
    "version": "string"
  },
  "messages": [                     // Ordered message array
    {
      "role": "user|assistant|system|tool",
      "content": "string|object",
      "timestamp": "ISO8601",
      "metadata": {}
    }
  ]
}
```

### 3.2 Optional Fields

```json
{
  "servers": [{                     // MCP servers involved
    "name": "string",
    "url": "string",
    "capabilities": ["resources", "prompts", "tools"]
  }],
  "tools": [{                       // Tools available/used
    "name": "string",
    "description": "string",
    "input_schema": {},
    "invocations": [{
      "timestamp": "ISO8601",
      "input": {},
      "output": {},
      "duration_ms": 0
    }]
  }],
  "resources": [{                   // Resources accessed
    "uri": "string",
    "name": "string",
    "mimeType": "string",
    "accessed_at": "ISO8601"
  }],
  "context": {                      // Session context
    "system_prompt": "string",
    "temperature": 0.0,
    "max_tokens": 0
  },
  "artifacts": [{                   // Produced outputs
    "type": "string",
    "content": "string|object",
    "created_at": "ISO8601"
  }]
}
```

---

## 4. Party Representation

MCP sessions involve non-human participants that require party representation:

### 4.1 AI Agent Party

```json
{
  "name": "Claude",
  "role": "assistant",
  "meta": {
    "type": "ai_agent",
    "provider": "anthropic",
    "model": "claude-4-opus",
    "model_version": "2026-01",
    "capabilities": ["text", "vision", "tool_use"]
  }
}
```

### 4.2 Tool/Service Party

```json
{
  "name": "Notion MCP Server",
  "role": "tool_provider",
  "meta": {
    "type": "mcp_server",
    "url": "https://mcp.notion.com/sse",
    "tools_provided": ["search", "create_page", "update_page"]
  }
}
```

### 4.3 Considerations

- Current vCon spec notes parties should include a "natural person" for data protection purposes
- AI agents and tools may need distinct party type indicators
- The `meta` field on parties can accommodate extended attributes
- Consider privacy implications of exposing tool/model details

---

## 5. Implementation Plan

### Phase 1: Define Media Type (1-2 months)

1. Draft `application/vnd.mcp+json` media type specification
2. Define core MCP session JSON schema
3. Register with IANA

### Phase 2: Extension Specification (2-3 months)

1. Write I-D for vCon MCP Session Extension
2. Define dialog type "mcp_session"
3. Define analysis types for MCP-derived data
4. Specify party extensions for AI agents
5. Submit to vCon working group

### Phase 3: Reference Implementation (2-3 months)

1. Build MCP-to-vCon converter library
2. Implement vCon signing for MCP sessions
3. Create validation tools
4. Test with major MCP implementations

### Phase 4: Integration with Lifecycle (ongoing)

1. Integrate with SCITT for provenance tracking
2. Define consent management for AI conversations
3. Address data subject rights for AI-generated content

---

## 6. Open Questions

1. **Tool output ownership**: Who "owns" content generated by tools? How does this affect vCon party attribution?

2. **Session boundaries**: How do we handle multi-session conversations where context persists?

3. **Streaming sessions**: MCP supports streaming; how do we capture incomplete/ongoing sessions?

4. **Confidentiality**: Should tool definitions and system prompts be redactable?

5. **Chain of custody**: When an AI agent invokes another AI agent (A2A), how deep should the vCon capture go?

6. **Analysis scope**: Can traditional vCon analysis (sentiment, summary) apply meaningfully to MCP sessions?

---

## 7. Recommendations

### Immediate (0-3 months)
- Adopt **Option B (Attachment)** for quick implementation
- Begin drafting MCP session content schema
- Prototype with existing vCon libraries

### Short-term (3-6 months)
- Develop **Option C (Extension)** specification
- Register extension with IANA
- Implement reference library

### Medium-term (6-12 months)
- Move to **Option E (Hybrid)** approach
- Integrate with vCon lifecycle (SCITT)
- Develop analysis schemas for MCP sessions

---

## 8. References

- draft-ietf-vcon-vcon-core-01: vCon Core Specification
- draft-ietf-vcon-overview-01: vCon Overview
- draft-howe-vcon-lifecycle-00: vCon Lifecycle Management
- draft-howe-vcon-wtf-extension-01: WTF Transcription Extension (pattern)
- draft-howe-vcon-lawful-basis-00: Lawful Basis Extension (pattern)
- draft-rosenberg-ai-protocols-00: AI Agent Protocols Framework
- Model Context Protocol Specification (modelcontextprotocol.io)
