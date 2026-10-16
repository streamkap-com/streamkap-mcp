# Streamkap MCP

<img src="logo.svg" alt="Streamkap" width="64" align="right">

Build, monitor, and troubleshoot real-time CDC pipelines from your AI agent.

[Streamkap](https://streamkap.com) is the developer platform for real-time data. It uses change data capture (CDC) to stream changes from your databases to apps, warehouses, and AI agents the moment they happen. The hosted Streamkap MCP server lets your agent operate all of it in natural language:

- **Check pipeline health:** "Which pipelines are failing, and what are their latest errors?"
- **Build a pipeline:** "Connect my Postgres orders table to Snowflake with incremental updates."
- **Diagnose latency:** "End-to-end lag on prod-events jumped this morning. What changed?"
- **Triage dead-letter queues:** "Find every DLQ with messages in the last hour, group by error class, and suggest fixes."

This repository holds the plugin and extension manifests for each AI client, plus the Streamkap agent skill. The server itself is hosted at `https://mcp.streamkap.com/mcp`: there is nothing to run, and you sign in with your Streamkap account on first use.

## Install

### Claude

Claude.ai and Claude Desktop: add **Streamkap** from **Settings → Connectors**.

Claude Code:

```
/plugin marketplace add streamkap-com/streamkap-mcp
/plugin install streamkap@streamkap-mcp
```

### Codex

```bash
codex plugin marketplace add streamkap-com/streamkap-mcp
codex plugin add streamkap@streamkap-mcp
```

### Cursor

[![Add to Cursor](https://cursor.com/deeplink/mcp-install-dark.svg)](https://cursor.com/link/mcp/install?name=streamkap&config=eyJ1cmwiOiJodHRwczovL21jcC5zdHJlYW1rYXAuY29tL21jcCJ9)

### VS Code

[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Streamkap-0098FF?style=flat-square&logo=visualstudiocode)](https://insiders.vscode.dev/redirect/mcp/install?name=streamkap&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fmcp.streamkap.com%2Fmcp%22%7D)

### Gemini CLI

```bash
gemini extensions install https://github.com/streamkap-com/streamkap-mcp
```

### Kiro

**Powers → Add Custom Power → Import from GitHub**, then enter `https://github.com/streamkap-com/streamkap-mcp`.

### Agent skill only

```bash
npx skills add streamkap-com/streamkap-mcp
```

### Any other MCP client

```json
{
  "mcpServers": {
    "streamkap": {
      "type": "http",
      "url": "https://mcp.streamkap.com/mcp"
    }
  }
}
```

## Authentication

By default your client signs you in through OAuth with your Streamkap account, and every call is scoped to the Streamkap tenant you choose. Destructive operations (deletes and resets) are off by default for OAuth sessions.

For direct Kafka produce and consume, or for headless agents and CI, connect with a [Project Key](https://docs.streamkap.com/project-keys) sent in the `X-Streamkap-Project-Key` header instead.

## What's in this repository

| Path | For |
|---|---|
| `plugin.json`, `mcp.json` | [Agent Plugins](https://agent-plugins.org) clients: Kiro, VS Code, Codex |
| `.claude-plugin/`, `.mcp.json` | Claude Code plugin and marketplace |
| `.codex-plugin/` | Codex and the ChatGPT plugin directory |
| `.cursor-plugin/` | Cursor Marketplace |
| `gemini-extension.json`, `GEMINI.md` | Gemini CLI extension |
| `skills/streamkap/` | The Streamkap agent skill |

The skill and the version number come from the [`@streamkap/tools`](https://www.npmjs.com/package/@streamkap/tools) npm package, and a daily workflow opens a pull request when it publishes a new version.

## Links

- Documentation: [docs.streamkap.com/mcp-server](https://docs.streamkap.com/mcp-server)
- Support: [support@streamkap.com](mailto:support@streamkap.com) · [streamkap.com/contact-us](https://streamkap.com/contact-us)
- [Privacy Policy](https://streamkap.com/privacy) · [Terms of Service](https://streamkap.com/terms)

## License

[MIT](LICENSE)
