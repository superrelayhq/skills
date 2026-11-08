# Superrelay agent skills

Agent skills for [Superrelay](https://superrelay.com), the support platform with live chat, an email and WhatsApp inbox, AI agents, a help center, a feature request board and a changelog. They teach Claude Code, Codex, Cursor and other coding agents to install Superrelay in your app and to work in your support workspace.

```bash
npx skills add superrelayhq/skills
```

Install one skill with `--skill <name>`, or for every project with `--global`.

| Skill | What it does |
| --- | --- |
| [`superrelay`](skills/superrelay/SKILL.md) | Read and answer conversations, triage the inbox, look up customers, manage feature requests, the changelog and help center articles through the `relay` CLI, the MCP server or the REST API |
| [`superrelay-widget`](skills/superrelay-widget/SKILL.md) | Add the chat widget to a website or web app, identify signed-in users and sign their identity on the server |
| [`superrelay-docs`](skills/superrelay-docs/SKILL.md) | Keep the help center in your repository as Markdown: pull, write, check, diff and push articles |
| [`superrelay-release`](skills/superrelay-release/SKILL.md) | Turn the commits since the last tag into a changelog entry, link the feature requests it ships and publish it |

## What you need

- A Superrelay workspace: [app.superrelay.com](https://app.superrelay.com).
- For the workspace skills, an API key from **Settings → API Keys** (`rl_live_…`) for the [`relay` CLI](https://www.npmjs.com/package/@superrelay/cli), or the MCP server connected to your agent:

  ```bash
  claude mcp add --transport http superrelay https://api.superrelay.com/v1/mcp
  ```

The skills ask before anything a customer would see: replies, published articles and changelog entries.

## Docs

- [Superrelay docs](https://docs.superrelay.com)
- [CLI](https://docs.superrelay.com/docs/api-reference/cli)
- [REST API](https://docs.superrelay.com/docs/api-reference)
- [MCP server](https://docs.superrelay.com/docs/implementation/mcp#superrelay-as-an-mcp-server)

## License

MIT
