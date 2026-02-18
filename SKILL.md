---
name: launchthatbot-import
version: 1.2.0
description: Export your OpenClaw agent config, memory, skills, and encrypted secrets to a LaunchThatBot deployment. One-time, end-to-end encrypted transfer -- LaunchThatBot never sees your raw API keys.
author: LaunchThatBot
homepage: https://launchthatbot.com
requires:
  mcp: launchthatbot
metadata:
  {
    "openclaw":
      {
        "emoji": "🚀",
        "requires": { "bins": [], "env": [], "config": [] },
      },
  }
---

# LaunchThatBot Agent Import

This skill lets you export your current OpenClaw agent configuration to a LaunchThatBot deployment. Your secrets (API keys, tokens) are encrypted before leaving this machine and are decrypted only inside your new container. LaunchThatBot never has access to the raw values.

## How It Works

1. The user generates a **one-time import token** from the LaunchThatBot dashboard
2. You call LaunchThatBot MCP through `mcporter` (`import_handshake`) to fetch the RSA public key automatically
3. You collect the agent's config files, memory, skills, and environment secrets
4. Secrets are encrypted with the RSA public key -- only the target container can decrypt them
5. Everything is sent through LaunchThatBot MCP via `mcporter` `import_push` (which forwards to LaunchThatBot import API)
6. The token expires after 15 minutes and can only be used once
7. After import, the target container is automatically restarted to pick up the new config

## Security Model

This skill is built around the question: "Why would I share my API keys with a platform I have never heard of?"

The answer: **you are not sharing them with us.**

- **Your secrets never touch the LaunchThatBot database.** They pass through the API as an opaque, RSA-encrypted blob. Only the target container holds the private key.
- **The import token is single-use and time-limited.** It expires in 15 minutes and is consumed on first use. Even if intercepted, it cannot be replayed.
- **All communication happens over HTTPS** with an additional layer of RSA-OAEP encryption on secrets.
- **After import, your secrets live in your own container's `.env` file** on your VPS. You can SSH in and verify them.

## When the User Asks to Export

When the user says something like "export my config to LaunchThatBot" or "migrate to LaunchThatBot", follow these steps:

## MCP + mcporter Prerequisite

This skill is **mcporter-first** for OpenClaw/Pi compatibility.

Before running this flow, verify prerequisites in this order:

1. Check `mcporter` is runnable:

```bash
mcporter --version || npx -y mcporter --version
```

If this fails, tell the user:
- "`mcporter` is required for this skill. Please enable `npx mcporter` (or install/configure mcporter), then run import again."

2. Check LaunchThatBot MCP is configured and discoverable:

```bash
(mcporter list || npx -y mcporter list)
(mcporter list launchthatbot --schema || npx -y mcporter list launchthatbot --schema)
```

If `launchthatbot` is not available, attempt automated setup (if environment allows) and then re-check. If automation is blocked, ask the user to configure LaunchThatBot MCP manually.

Recommended MCP config:

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

3. Validate import tools are available before continuing:

```bash
(mcporter list launchthatbot --schema || npx -y mcporter list launchthatbot --schema)
```

Confirm `import_handshake` and `import_push` exist.

### Step 1: Collect Information from the User

Ask the user for:

1. **Import Token** -- a 64-character hex string from the LaunchThatBot dashboard
2. **API URL** -- the LaunchThatBot API URL (default: `https://api.ltb.it.com`)

The user gets the import token by clicking **Import Agent** on their agent's detail page in the LaunchThatBot dashboard.

### Step 2: Fetch the Public Key (Handshake via mcporter)

Call LaunchThatBot MCP through `mcporter`:

```
npx -y mcporter call launchthatbot.import_handshake \
  importToken:"<importToken>" \
  apiUrl:"https://api.ltb.it.com"
```

Response:

```json
{
  "publicKey": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----",
  "agentName": "My Agent",
  "expiresAt": 1708000000000
}
```

If the response is a 401, the token is invalid, expired, or already used. Ask the user to generate a new one.

### Step 3: Collect Agent Data

Gather the following from the local filesystem:

**soul.md** (agent personality/instructions):

- Check `/home/node/.openclaw/config/agents/*/soul.md`
- Or wherever the agent's soul.md is located

**Memory files**:

- Check `/home/node/.openclaw/memory/`
- Include all `.md` and `.json` files

**Skills**:

- Check `/home/node/.openclaw/skills/`
- Include all skill directories and their contents
- Exclude this skill (`launchthatbot-import`) from the export

**Environment secrets** (`.env` file):

- Read `/home/node/.openclaw/.env`
- Parse each `KEY=VALUE` line
- These will be encrypted before sending

### Step 4: Confirm with the User

Before sending anything, show the user a summary:

- Number of config files found
- Number of memory files found
- Number of skill directories found
- Number of environment variables found (show keys only, never values)

Ask the user to confirm they want to proceed.

### Step 5: Encrypt Secrets

For each secret from the `.env` file, encrypt the VALUE using the RSA public key fetched from the handshake endpoint. Use RSA-OAEP with SHA-256 padding.

```javascript
const crypto = require("crypto");

function encryptSecret(value, publicKeyPem) {
  const encrypted = crypto.publicEncrypt(
    {
      key: publicKeyPem,
      padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
      oaepHash: "sha256",
    },
    Buffer.from(value, "utf8"),
  );
  return encrypted.toString("base64");
}
```

If encryption fails, verify the public key is a valid PEM string starting with `-----BEGIN PUBLIC KEY-----`.

### Step 6: Send the Payload (via mcporter)

Send everything via LaunchThatBot MCP through `mcporter`:

```
npx -y mcporter call launchthatbot.import_push --args '{
  "importToken": "<importToken>",
  "apiUrl": "https://api.ltb.it.com",
  "payload": {
    "config": {
      "soulMd": "<contents of soul.md>",
      "memory": [
        { "filename": "MEMORY.md", "content": "<file contents>" },
        { "filename": "daily-log.json", "content": "<file contents>" }
      ],
      "skills": [
        { "path": "web-search/SKILL.md", "content": "<file contents>" },
        { "path": "email-sender/SKILL.md", "content": "<file contents>" }
      ]
    },
    "encryptedSecrets": [
      { "key": "OPENAI_API_KEY", "ciphertextB64": "<base64 encrypted value>" },
      { "key": "ANTHROPIC_API_KEY", "ciphertextB64": "<base64 encrypted value>" }
    ]
  }
}'
```

### Step 7: Report Results

If successful, tell the user:

- How many config files were written to the new container
- How many secrets were imported
- That the container was automatically restarted to pick up the new configuration
- That they can verify their secrets by SSHing into their VPS or checking the Convex dashboard
- That they can now safely shut down this old instance

If it fails, report the error and suggest generating a new import token from the LaunchThatBot dashboard.

## Important Exclusions

Do NOT include in the export:

- This skill itself (`launchthatbot-import/`)
- The `convex-backend` skill (it will be re-provisioned by LaunchThatBot)
- Any `.git` directories
- `node_modules` directories
- Temporary files or caches
- Lock files (`package-lock.json`, `pnpm-lock.yaml`, etc.)

## Error Handling

| Error                     | What to Do                                                                                       |
| ------------------------- | ------------------------------------------------------------------------------------------------ |
| 401 Unauthorized          | Token is invalid, expired, or already used. Generate a new one from the LaunchThatBot dashboard. |
| 400 Bad Request           | Check the payload format matches the schema above.                                               |
| 500 Internal Server Error | Server-side issue. Wait a moment and try again, or contact support on the LaunchThatBot Discord. |
| Network error             | Check internet connectivity. Verify the API URL is correct (default: `https://api.ltb.it.com`).  |

## Support

- **Discord**: [LaunchThatBot Community](https://discord.gg/launchthatbot)
- **Website**: [launchthatbot.com](https://launchthatbot.com)
- **Blog**: [Import Path Architecture](https://launchthatbot.com/blog/bringing-your-openclaw-setup-to-launchthatbot)
