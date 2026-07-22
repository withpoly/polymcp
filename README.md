# Poly MCP Plugin

Poly is a cloud file system with intelligent, embedding-powered search — store your files, find them by meaning, and chat with agents about them. This repo contains the official **Poly plugin**: the Poly MCP server configuration plus the `poly-cli` agent skill that teaches AI agents how to search, read, create, update, and organize files across your personal and shared drives.

Agents get two ways in:

- **Poly desktop CLI** (`poly`) — preferred whenever the agent runs on your machine. Ships with the Poly desktop app; talks to the local cache, works offline, and supports the full feature set including version history and rollbacks.
- **Poly MCP server** (`https://mcp.poly.app/mcp`) — a remote, more limited version of the CLI for web and cloud agents.

The [`poly-cli` skill](skills/poly-cli/SKILL.md) tells the agent how to pick between them and how to use each one.

## Installation & Setup

### ChatGPT

Install Poly through the plugin marketplace:

1. Go to **Plugins**.
2. Click the chevron dropdown menu next to **Create**.
3. Add a marketplace with the URL: `git@github.com:withpoly/polymcp.git` (under "Source", leave other fields blank)

This installs the `poly-official` marketplace with a single app, **Poly**, which you should now be able to search for and add.

> [!NOTE]
> You may need to restart the ChatGPT app for the Poly app to be detected correctly.



### Codex

Ask Codex to add Poly as a skill and point it at this repository:

```
Add the Poly plugin as a skill from https://github.com/withpoly/polymcp
```

This repo follows the plugin marketplace structure (see `.agents/plugins/marketplace.json`), so it can also be added as a plugin marketplace with Poly as its single plugin.

### Claude Desktop

Use the Poly Desktop Extension: open the Poly desktop app, go to the **Downloads** menu, and install the extension for Claude Desktop. This bundles the MCP connection and the skill with no manual configuration.

### Claude Code

Add this repo as a plugin marketplace and install the plugin:

```bash
claude plugin marketplace add withpoly/polymcp
claude plugin install poly@poly
```

<details>
<summary>Manual MCP-only setup</summary>

```bash
claude mcp add --transport http poly https://mcp.poly.app/mcp
```

</details>

### Any MCP-native client

Any client that supports streamable HTTP can connect to the Poly MCP server directly:

- **Server URL:** `https://mcp.poly.app/mcp` (streamable HTTP)

Typical configuration:

```json
{
  "mcpServers": {
    "poly": {
      "type": "http",
      "url": "https://mcp.poly.app/mcp"
    }
  }
}
```

Authentication is handled via OAuth on first connection.

### Gemini CLI

```bash
gemini extensions install https://github.com/withpoly/polymcp
```

Then authenticate inside the CLI with `/mcp auth poly`.

## Using the plugin

Once installed, just ask your agent about your files:

- "Search my Poly files for last quarter's planning docs"
- "Create a folder in my Poly drive and move these files into it"
- "Summarize the contents of the design folder in our shared drive"

If the agent runs on your desktop, make sure the Poly app is running and the CLI is installed (Poly app → **Downloads** menu). The agent will verify its connection with `poly app root` before doing real work.

### Desktop vs. remote capabilities

| Capability | Desktop CLI | Remote MCP |
|---|---|---|
| Search, list, read extracted text | ✅ | ✅ |
| Create, update, move, delete | ✅ | ✅ |
| Raw media access (audio/video/PDF bytes) | ✅ | ❌ (extracted text only) |
| Version history, restore, rollback | ✅ | ❌ |
| Deep folder copies | ✅ | ❌ |
| Bulk organization (dedupe, cleanup) | ✅ | ⚠️ suboptimal — prefer the desktop app or a desktop agent |

For heavy or complex operations, prefer the Poly desktop app or a desktop agent with CLI access (e.g. Claude CoWork, ChatGPT Work, or a local harness like Claude Code).

## Repo layout

- [`.mcp.json`](.mcp.json) — MCP server configuration pointing at `https://mcp.poly.app/mcp`
- [`server.json`](server.json) — MCP registry server manifest
- [`skills/poly-cli/`](skills/poly-cli/) — the `poly-cli` agent skill
- [`.claude-plugin/`](.claude-plugin/), [`.codex-plugin/`](.codex-plugin/), [`.cursor-plugin/`](.cursor-plugin/), [`.github/plugin/`](.github/plugin/) — per-client plugin manifests
- [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json), [`.agents/plugins/marketplace.json`](.agents/plugins/marketplace.json) — the Poly plugin marketplace, with this plugin as its single entry
- [`assets/`](assets/) — Poly logos and icons
