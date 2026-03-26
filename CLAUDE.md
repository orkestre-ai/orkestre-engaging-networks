# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This is the **public** Claude Code plugin marketplace for distributing production-ready Engaging Networks API integration skills.

## Repository Structure

This marketplace uses `strict: false` mode - the marketplace entry in `marketplace.json` defines all components directly, with no per-plugin `plugin.json` files.

```
.claude-plugin/
  marketplace.json    # Main marketplace manifest (required) — name: orkestre-engaging-networks
.references/
  engaging-networks-rest-api.json  # OpenAPI 3.1.0 spec for EN REST API v6.0.0
skills/               # Skills distributed via the marketplace
  engaging-networks-api/       # REST API integration (session-based auth, 44 endpoints)
```

## Skills

| Skill | Purpose |
|-------|---------|
| `engaging-networks-api` | Build, debug, optimize, and deploy EN REST API integrations (44 endpoints, 8 service groups) |

## Marketplace Schema

The `.claude-plugin/marketplace.json` file must include:
- `name`: Marketplace identifier (kebab-case) — this repo uses `orkestre-engaging-networks`
- `owner`: Object with `name` and optionally `email`
- `plugins`: Array of plugin entries

This repo uses `strict: false` - all component definitions live in `marketplace.json`.

## Key Gotchas

- **Plugin caching**: Installed plugins are copied to `~/.claude/plugins/cache`. They cannot reference files outside their directory with `../` paths.
- **`${CLAUDE_PLUGIN_ROOT}`**: Use this variable in hooks and MCP/LSP server configs to reference files within the plugin's install directory.
- **Version conflicts**: Avoid setting version in both `plugin.json` and `marketplace.json`. The plugin manifest always wins silently.

## Installation

```bash
# Add this marketplace
/plugin marketplace add orkestre-ai/orkestre-engaging-networks

# Install the plugin
/plugin install ork-engaging-networks@orkestre-engaging-networks
```

## Auto-Install (Project Settings)

Add to your project's `.claude/settings.json`:
```json
{
  "extraKnownMarketplaces": {
    "orkestre-engaging-networks": {
      "source": { "source": "github", "repo": "orkestre-ai/orkestre-engaging-networks" }
    }
  },
  "enabledPlugins": {
    "ork-engaging-networks@orkestre-engaging-networks": true
  }
}
```

Users refresh their local copy with `/plugin marketplace update`.
