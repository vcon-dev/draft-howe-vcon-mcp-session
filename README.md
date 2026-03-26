# draft-howe-vcon-mcp-session

This repository contains the working draft for the vCon Model Context Protocol (MCP) Session Extension.

## Document

**draft-howe-vcon-mcp-session-00**: Defines how to capture MCP sessions (human-AI conversations with tool invocations) within vCon containers.

## Overview

The MCP Session extension provides a standardized framework for:

- Capturing human-AI conversations in vCon format
- Recording tool invocations and their results
- Representing AI agents and MCP servers as conversation parties
- Storing artifacts produced during AI interactions
- Supporting analysis and audit of AI agent behavior

## Approach

This extension uses a **hybrid approach** combining:

1. **Dialog Objects** (`type: "mcp_session"`): Raw session transcripts
2. **Party Objects**: Extended to represent AI agents and tool services
3. **Analysis Objects**: Derived insights from MCP sessions
4. **Attachment Objects**: Tool outputs and artifacts

## Status

- **Version**: 00 (Initial Draft)
- **Status**: Work in Progress
- **Working Group**: vCon

## Files

- `draft-howe-vcon-mcp-session-00.md` - Main specification (Markdown)
- `examples/` - Example vCons with MCP sessions (coming soon)

## Related Specifications

- [draft-ietf-vcon-vcon-core](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-core/) - vCon Core
- [draft-howe-vcon-wtf-extension](https://datatracker.ietf.org/doc/draft-howe-vcon-wtf-extension/) - WTF Transcription Extension (pattern reference)
- [draft-howe-vcon-lawful-basis](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) - Lawful Basis Extension (pattern reference)
- [Model Context Protocol](https://modelcontextprotocol.io/) - MCP Specification

## Contributing

Discussion takes place on the vCon Working Group mailing list: vcon@ietf.org

## License

See LICENSE file for terms.
