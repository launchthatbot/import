# @launchthatbot/import

OpenClaw skill package that exports an agent's config, memory, skills, and encrypted secrets to a LaunchThatBot deployment.

This skill expects the `launchthatbot` MCP server to be available and uses MCP tools (`import_handshake`, `import_push`) for transport, typically via `mcporter`.

## Install

```bash
npx clawhub@latest install launchthatbot-import
```

Or copy `SKILL.md` manually into your OpenClaw skill folder.

## MCP requirement

Configure the LaunchThatBot MCP server in your MCP client:

```json
{
  "mcpServers": {
    "launchthatbot": {
      "command": "npx",
      "args": ["-y", "@launchthatbot/mcp-server"],
      "env": {
        "LAUNCHTHATBOT_API_KEY": "ltb_sk_..."
      }
    }
  }
}
```

## mcporter preflight

Recommended checks before running import:

```bash
mcporter --version || npx -y mcporter --version
mcporter list || npx -y mcporter list
mcporter list launchthatbot --schema || npx -y mcporter list launchthatbot --schema
```

If `mcporter` or `launchthatbot` MCP is missing, enable `npx mcporter` (or install/configure mcporter), then retry import.

## Security model

- Secrets are encrypted on the source machine using RSA-OAEP (SHA-256).
- LaunchThatBot receives encrypted secret payloads and cannot decrypt them.
- Import tokens are single-use and time-limited.

## Support

- Website: https://launchthatbot.com
- Discord: https://discord.gg/launchthatbot
