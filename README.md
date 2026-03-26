# Brocode Claude Dev Skills

A catalog of all Claude Code skills available on this system — custom, plugin-based, and built-in.

---

## Custom Global Skills

Skills installed at `~/.claude/skills/`.

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 1 | **deploy-frappe** | Deploy Frappe Framework, ERPNext, or any Frappe app on a server or Proxmox VM. Covers full stack installation — system dependencies, MariaDB, Node.js, bench setup, Python 3.12 fixes, site creation, and networking. | Custom (Brocode) |
| 2 | **proxmox-vm** | Create, update, delete, or manage VMs on Proxmox. Covers cloud-init provisioning, OS setup, guest agent installation, networking, and software deployment — all without manual console interaction. | Custom (Brocode) |
| 3 | **setup-mcp** | Install, configure, or troubleshoot the Proxmox MCP server for Claude Code. Covers cloning the repo, creating API tokens, configuring `.mcp.json`, and verifying the connection. | Custom (Brocode) |

---

## Plugin-Based Skills

Skills installed via Claude Code plugins at `~/.claude/plugins/marketplaces/`.

### claude-code-setup Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 4 | **claude-automation-recommender** | Analyze a codebase and recommend Claude Code automations (hooks, subagents, skills, plugins, MCP servers). Use when optimizing Claude Code setup or first configuring Claude Code for a project. | claude-code-setup plugin |

### mcp-server-dev Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 5 | **build-mcp-server** | Build, create, or scaffold an MCP server. Entry point for MCP server development — wrap an API for Claude, expose tools, or make an MCP integration. | mcp-server-dev plugin |
| 6 | **build-mcp-app** | Add interactive UI/widgets to an MCP server — forms, pickers, dashboards, confirmation dialogs rendered inline in chat. Use after `build-mcp-server` or when UI widgets are the goal. | mcp-server-dev plugin |
| 7 | **build-mcpb** | Package and bundle an MCP server into a distributable `.mcpb` file. Ships Node/Python runtime so end users don't need dev tooling installed. | mcp-server-dev plugin |

### claude-md-management Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 8 | **claude-md-improver** | Audit and improve CLAUDE.md files in repositories. Scans for all CLAUDE.md files, evaluates quality against templates, outputs a quality report, then makes targeted updates. | claude-md-management plugin |

### playground Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 9 | **playground** | Create interactive HTML playgrounds — self-contained single-file explorers with live preview and configurable controls. Great for visual prototyping. | playground plugin |

### plugin-dev Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 10 | **command-development** | Create, define, and organize custom slash commands with frontmatter, arguments, and file references. | plugin-dev plugin |
| 11 | **skill-development** | Create, structure, and improve Claude Code skills for plugins. | plugin-dev plugin |
| 12 | **plugin-settings** | Configure plugin settings and preferences. | plugin-dev plugin |
| 13 | **plugin-structure** | Organize and scaffold plugin directory structure. | plugin-dev plugin |
| 14 | **hook-development** | Create Claude Code hooks — automated shell commands triggered by tool events. | plugin-dev plugin |
| 15 | **mcp-integration** | Integrate MCP servers into plugins. | plugin-dev plugin |
| 16 | **agent-development** | Develop custom subagents within plugins. | plugin-dev plugin |

### skill-creator Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 17 | **skill-creator** | Create new skills from scratch, modify existing skills, run evals, benchmark performance with variance analysis, and optimize skill descriptions for better triggering accuracy. | skill-creator plugin |

### math-olympiad Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 18 | **math-olympiad** | Solve competition math problems (IMO, Putnam, USAMO, AIME) with adversarial verification that catches errors self-verification misses. | math-olympiad plugin |

### frontend-design Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 19 | **frontend-design** | Create distinctive, production-grade frontend interfaces with high design quality — web components, pages, and applications. | frontend-design plugin |

### hookify Plugin

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 20 | **writing-rules** | Write and configure Claude Code writing rules and hooks. | hookify plugin |

### Messaging Plugins (Discord, Telegram, iMessage)

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 21 | **discord:access** | Access Discord channels, messages, and servers from Claude Code. | discord plugin |
| 22 | **discord:configure** | Set up and configure the Discord integration. | discord plugin |
| 23 | **telegram:access** | Access Telegram chats and messages from Claude Code. | telegram plugin |
| 24 | **telegram:configure** | Set up and configure the Telegram integration. | telegram plugin |
| 25 | **imessage:access** | Access iMessage conversations from Claude Code. | imessage plugin |
| 26 | **imessage:configure** | Set up and configure the iMessage integration. | imessage plugin |

### example-plugin (Templates)

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 27 | **example-skill** | Reference template for building new skills. | example-plugin |
| 28 | **example-command** | Reference template for building new commands. | example-plugin |

---

## Built-in System Skills

Shipped with Claude Code itself.

| # | Skill | Use Case | Creator |
|---|-------|----------|---------|
| 29 | **update-config** | Configure the Claude Code harness — permissions, env vars, hooks, and `settings.json`. Use for automated behaviors ("whenever X do Y"), adding permissions, or setting environment variables. | Anthropic (built-in) |
| 30 | **keybindings-help** | Customize keyboard shortcuts, rebind keys, add chord bindings, or modify `~/.claude/keybindings.json`. | Anthropic (built-in) |
| 31 | **simplify** | Review changed code for reuse, quality, and efficiency, then fix any issues found. | Anthropic (built-in) |
| 32 | **loop** | Run a prompt or slash command on a recurring interval (e.g., `/loop 5m /foo`). Use for polling, recurring checks, or repeated tasks. | Anthropic (built-in) |
| 33 | **schedule** | Create, update, list, or run scheduled remote agents (triggers) that execute on a cron schedule. | Anthropic (built-in) |
| 34 | **claude-api** | Build apps with the Claude API or Anthropic SDK. Triggers when code imports `anthropic` or `@anthropic-ai/sdk`. | Anthropic (built-in) |

---

## Summary

| Category | Count |
|----------|-------|
| Custom Global Skills | 3 |
| Plugin-Based Skills | 25 |
| Built-in System Skills | 6 |
| **Total** | **34** |
