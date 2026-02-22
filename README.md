# Lark.sh Plugin for Claude Code

Official [Lark.sh](https://lark.sh) plugin for Claude Code. Gives Claude full knowledge of the Lark CLI, data API, project management, security rules, and common patterns.

## Install

Add the marketplace and install the plugin:

```
/plugin marketplace add lark-sh/lark-skill
/plugin install lark@lark-tools
```

Then use `/lark` in any conversation to activate the skill, or Claude will load it automatically when you're working with Lark.

## What's included

- Full CLI command reference (`lark data`, `lark projects`, `lark databases`, `lark rules`, etc.)
- REST data API format and authentication
- Admin API endpoints
- Common patterns: backup/restore, seeding data, watch scripts, CI/CD usage
- ID formats and conventions

## Links

- Docs: https://docs.larksh.com
- Dashboard: https://dashboard.lark.sh
- MCP server: https://docs.larksh.com/mcp
