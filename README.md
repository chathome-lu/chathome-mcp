# ChatHome MCP

Connect an AI assistant to live property search across the Grande Région and source-backed Luxembourg market information.

- Endpoint: <https://chathome.lu/api/mcp>
- Setup: <https://chathome.lu/en/mcp>
- Discovery: <https://chathome.lu/.well-known/mcp.json>
- Health: <https://chathome.lu/api/mcp/health>
- Public REST OpenAPI 3.1: <https://chathome.lu/openapi.json>
- Official registry name: `lu.chathome/chathome`

This repository contains the public interface, tool schemas and integration documentation. It does not contain the hosted server implementation or private application source. The JSON snapshots describe the live endpoint at the time in `snapshot.json`; the hosted service can change independently.

## Authentication and safety

Discovery (`initialize`, `tools/list`, prompts and public resources) works without authentication. Seven public tools also execute anonymously, within per-IP and global hourly limits:

`search_vibe`, `search_rentals`, `search_for_sale`, `render_listing_cards`, `get_listing`, `compare_listings`, `get_market_context`.

All anonymous tools declare `readOnlyHint: true` and `destructiveHint: false`. The full catalogue has 15 tools: 13 reads and two writes. `submit_inquiry` and `analyze_lease` require authentication. Lease analysis also requires an OAuth user. User-specific inquiry status requires authentication. Public discovery of a tool is not permission to execute it anonymously.

OAuth 2.1 with PKCE and dynamic registration is supported. Scripts can use `X-API-Key`; obtain a key from your own ChatHome account. Never put a real key into a shared config or this repository.

The server uses stateless Streamable HTTP. Send POST with `Content-Type: application/json` and `Accept: application/json, text/event-stream`. GET returns 405 because the server does not open a server-initiated SSE stream; use the health URL for liveness checks.

## Client setup

Claude web and Desktop: add the endpoint under Settings → Connectors. For a local Desktop bridge, use `configs/claude_desktop_config.json`; it runs the external `mcp-remote` npm package and requires Node.js. Native remote connectors are the simpler setup.

Cursor: merge `configs/cursor.json` into `~/.cursor/mcp.json` or `.cursor/mcp.json`.

VS Code Copilot: merge `configs/vscode.json` into `.vscode/mcp.json`.

Raycast Pro: run **Install MCP Server**, choose HTTP, use the endpoint, and choose Dynamic OAuth when signing in. No Raycast Store extension is needed for this MCP connection.

Claude Code: `claude mcp add --transport http chathome https://chathome.lu/api/mcp`.

## Example prompts

- Find a quiet two-bedroom rental near Kirchberg below €2,000. Show the photos and source links, and state which wishes were not checked.
- Compare confirmed monthly costs for these listing IDs. Keep unknown charges visible.
- Show the source dates and price context for Hesperange and Bettembourg. Do not treat asking prices as completed sale prices.

Price estimates and commute tools require authentication. Tool availability and client plan requirements vary. There is no published ChatHome Custom GPT or Raycast Store extension claimed by this package.

## Files

- `tools.json`: live input/output schemas, annotations and per-tool auth declarations.
- `manifest.json`: live discovery metadata.
- `openapi.json`: live public REST schema, including the general MCP transport operation.
- `gpt-actions.json`: six named read operations for the GPT editor, without general MCP dispatch. Set authentication to None; use `gpt-instructions.txt` for the assistant instructions. The hosted schema is available at [`/.well-known/gpt-actions.json`](https://chathome.lu/.well-known/gpt-actions.json).
- `server.json`: draft update for the existing official registry record, including the proposed documentation repository in publisher metadata. It does not claim this repository contains the hosted implementation. Publish the repository before submitting this update.
- `snapshot.json`: observation time and endpoint.

## Setup references

- [Claude remote connectors](https://support.claude.com/en/articles/11503834-building-custom-connectors-via-remote-mcp-servers)
- [mcp-remote bridge](https://github.com/punkpeye/mcp-remote)
- [Cursor MCP](https://cursor.com/help/customization/mcp)
- [VS Code MCP configuration](https://code.visualstudio.com/docs/agents/reference/mcp-configuration)
- [Raycast MCP](https://manual.raycast.com/ai/model-context-protocol)
