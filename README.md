# @launchthatbot/import

OpenClaw skill package that exports an agent's config, memory, skills, and encrypted secrets to a LaunchThatBot deployment.

## Source of truth

- Monorepo package path: `packages/launchthatbot-import`
- Mirror repository: `https://github.com/launchthatbot/import`
- Sync workflow: `.github/workflows/sync-skill-mirrors.yml`

## Install

```bash
npx clawhub@latest install launchthatbot-import
```

Or copy `SKILL.md` manually into your OpenClaw skill folder.

## Security model

- Secrets are encrypted on the source machine using RSA-OAEP (SHA-256).
- LaunchThatBot receives encrypted secret payloads and cannot decrypt them.
- Import tokens are single-use and time-limited.

## Support

- Website: https://launchthatbot.com
- Discord: https://discord.gg/launchthatbot
