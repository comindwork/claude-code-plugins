# Comind.work Claude Plugins

Claude Code plugin marketplace for Comind.work.

## Install

```bash
claude plugin marketplace add comindwork/claude-code-plugins
claude plugin install comind-dev@comind-claude-code-plugins
claude plugin install comind-user@comind-claude-code-plugins
```

## Available plugins

| Plugin | Audience | Skills | Description |
|---|---|---|---|
| `comind-dev` | Developers | `/comind-dev:apps-dev`, `/comind-dev:apps-test`, `/comind-dev:plugins-dev` | App development, testing, plugin creation |
| `comind-user` | End-users, admins | `/comind-user:everyday`, `/comind-user:workspace-admin`, `/comind-user:system-admin` | Daily tasks, workspace and system management |

## Scope

By default installs globally (`user` scope). To install per-project instead:

```bash
claude plugin install comind-dev@comind-claude-code-plugins --scope project
```

To move from global to project scope:

```bash
claude plugin uninstall comind-dev --scope user
claude plugin install comind-dev@comind-claude-code-plugins --scope project
```

## Update

First refresh the marketplace, then update the plugin:

```bash
claude plugin marketplace update comind-claude-code-plugins
claude plugin update comind-dev@comind-claude-code-plugins --scope project
```

If installed at user scope, use `--scope user` instead.
