# Connecting Claude Code to a Legacy On-Prem Jira & Confluence (Server 7.6.13)

A minimal MCP (Model Context Protocol) setup that lets [Claude Code](https://claude.com/claude-code) search and read tickets/pages from a **self-hosted Jira Server / Confluence Server** instance — no Atlassian Cloud, no internet exposure required.

## Why this exists

Most "connect AI to Jira" guides assume Atlassian Cloud, which has an official MCP server and supports modern auth (API tokens, OAuth, PATs). On-prem **Jira Server 7.6.13** has none of that:

- Personal Access Tokens weren't introduced until Jira 8.14 — too old here.
- No OAuth app registration available.
- Only classic Basic Auth (username + password) against the REST API works.
- The instance only lives on the corporate LAN/VPN, never exposed to the internet.

This repo documents the working configuration so Claude can run JQL searches ("find my open tickets", "search tickets by keyword") and Confluence keyword search directly from a local dev machine, as long as that machine can already reach the Jira/Confluence URLs in a browser.

## How it works

```
Claude Code  --(stdio MCP)-->  mcp-atlassian (Python, local process)  --(HTTPS + Basic Auth)-->  Jira/Confluence Server REST API
```

- [`mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) is an open-source MCP server that wraps the Jira/Confluence REST APIs and exposes them as MCP tools (search, get issue, search pages, etc.). It supports Cloud, Server, and Data Center deployments.
- Because Server 7.6.13 has no PAT support, the `JIRA_API_TOKEN` / `CONFLUENCE_API_TOKEN` fields are simply used as the **password** in Basic Auth — that's how the underlying client library treats them when the target isn't `*.atlassian.net`.
- Credentials are never hardcoded in the committed config. `.mcp.json` pulls them from OS environment variables at launch time (`${JIRA_USERNAME}`, `${JIRA_API_TOKEN}`, etc.), so the file itself is safe to commit and share.

## Files

| File | Purpose |
|---|---|
| `.mcp.json` | Claude Code's project-scoped MCP server definition. Drop this in a repo root and Claude Code auto-detects it. References credentials via env-var interpolation — no secrets inside. |
| `mcp-atlassian.env` | Template for the alternative `--env-file` startup method (useful if you don't want to manage OS-level environment variables). **Never commit this with real values filled in.** |

## Setup

1. **Install the MCP server** (requires Python 3.10+):
   ```
   pip install mcp-atlassian
   ```
   Confirm it's on PATH: `where mcp-atlassian` (Windows) / `which mcp-atlassian` (macOS/Linux).

2. **Set credentials as environment variables** (used by `.mcp.json`):
   - `JIRA_USERNAME`, `JIRA_API_TOKEN`
   - `CONFLUENCE_USERNAME`, `CONFLUENCE_API_TOKEN`

   On Windows: System Properties → Environment Variables, or `setx JIRA_USERNAME "you"`.

3. **Fill in your actual Jira/Confluence base URLs** in `.mcp.json` (replace the `<Your Jira URL>` / `<Your Confluence URL>` placeholders).

4. **Drop `.mcp.json` into the root of any project** you open with Claude Code. On first launch it'll prompt you to approve the project-scoped MCP server.

5. Ask Claude things like:
   - "Search Jira for tickets assigned to me that are still open"
   - "Find tickets mentioning <keyword>"
   - "Search Confluence for documentation about <keyword>"
   - "Show me the full details of PROJ-1234"

## Security notes

- This setup only works because the machine running Claude Code is already on the same network/VPN as the Jira/Confluence server — no new network exposure is introduced.
- Basic Auth over plain HTTP would leak credentials on the wire; this only made sense here because the instance is served over HTTPS.
- Treat the filled-in `.env` file like any other secret: don't commit it, and consider OS-level file permissions if the machine is shared.

## Limitations

- Jira Server 7.6.13 is end-of-life and unsupported by Atlassian. This is a stopgap for legacy environments, not a recommended long-term architecture — migrating to Jira Cloud or Data Center with PAT/OAuth support would be the more secure path.
- Read-heavy use case (search/lookup). Write operations (creating/editing issues) are technically supported by `mcp-atlassian` but weren't the focus here and should be reviewed before enabling in a write-capable mode.
