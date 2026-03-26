# Orkestre Engaging Networks

Claude Code skills for building, debugging, and deploying Engaging Networks API integrations for nonprofit organizations.

## Skills

| Skill | Description |
|-------|-------------|
| **engaging-networks-api** | Full REST API integration (44 endpoints, 8 service groups). Session-based auth, TypeScript/Python clients, write safety guardrails, export jobs, marketing automations. |

## Installation

```bash
# Add the marketplace
/plugin marketplace add orkestre-ai/orkestre-engaging-networks

# Install the plugin
/plugin install ork-engaging-networks@orkestre-engaging-networks
```

**Auto-install** (add to your project's `.claude/settings.json`):

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

## Prerequisites

- **EN API User token** from EN Admin > Settings > API Users
- **IP whitelisting** configured for your server
- `.env` with `EN_API_USER_TOKEN` and `EN_REGION` (us, us2, ca, eu, or au)

## Quick Start

Just ask Claude anything about the Engaging Networks API:

- *"Build a TypeScript integration with EN to sync donor data"*
- *"Add recurring transaction tracking to my EN client"*
- *"Getting 401 errors from EN API, help me debug"*
- *"Set up a bulk data export from EN"*

The skill activates automatically based on your request.

## API Coverage

| Service Group | Endpoints | Description |
|---------------|-----------|-------------|
| Authentication | 3 | Session token management |
| Account | 1 | Audit log access |
| Page Services | 5 | Campaign pages, submissions, surveys |
| Page Components | 17 | Templates, code blocks, widgets, forms |
| Supporter Services | 17 | CRUD, queries, fields, import formats |
| Origin Source | 4 | Acquisition channel tracking |
| Export Job Services | 3 | Bulk data exports |
| Marketing Automation | 4 | Automated journeys and stats |

**44 endpoints** across 8 service groups, each classified as READ, WRITE, or DESTRUCTIVE with appropriate safety guardrails.

## Write Safety

The `engaging-networks-api` skill protects production donor data with a three-layer safety model:

1. **Risk classification** -- every write endpoint is tagged WRITE or DESTRUCTIVE
2. **Confirmation gates** -- Claude shows you the exact request and waits for approval before any write
3. **Dry-run pattern** -- generated code includes `dryRun` option to preview without executing

See `skills/engaging-networks-api/README.md` for full details.

## License

MIT
