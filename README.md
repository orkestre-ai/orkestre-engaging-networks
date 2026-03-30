```
    .::::            .::                        .::
  .::    .::         .::                        .::
.::        .::.: .:::.::  .::   .::     .:::: .:.: .:.: .:::   .::
.::        .:: .::   .:: .::  .:   .:: .::      .::   .::    .:   .::
.::        .:: .::   .:.::   .::::: .::  .:::   .::   .::   .::::: .::
  .::     .::  .::   .:: .:: .:            .::  .::   .::   .:
    .::::     .:::   .::  .::  .::::   .:: .::   .:: .:::     .::::

Agent skills for Engaging Networks API integrations.
Compatible with Claude Code, OpenAI Codex CLI, and any Agent Skills-compatible tool.
```

# Orkestre Engaging Networks

Agent skills for building, debugging, and deploying Engaging Networks API integrations for nonprofit organizations. Compatible with Claude Code, OpenAI Codex CLI, and any tool supporting the [Agent Skills specification](https://agentskills.io/specification).

## Skills

| Skill | Description |
|-------|-------------|
| **engaging-networks-api** | Full REST API integration (44 endpoints, 8 service groups). Session-based auth, TypeScript/Python clients, write safety guardrails, export jobs, marketing automations. |

## Installation

### Claude Code

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

### OpenAI Codex CLI

```bash
codex skills add orkestre-ai/orkestre-engaging-networks
```

### Other Tools

This repo follows the [Agent Skills specification](https://agentskills.io/specification). Any compatible tool can consume the skills in `skills/`.

## Prerequisites

- **EN API User token** from EN Admin > Settings > API Users
- **IP whitelisting** configured for your server
- `.env` with `EN_API_USER_TOKEN` and `EN_REGION` (us, us2, or ca — EU/AU clients use ca)

## Quick Start

Connect to a **sandbox or test EN account** for development (recommended). Just ask your AI coding assistant anything about the Engaging Networks API:

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

## Data Privacy & Intended Use

This skill is a **development assistant** — it helps you write code that integrates with the EN API. It is not designed as a live data operations proxy.

**When your agent queries the EN API, response data is processed by the model provider's infrastructure** (Anthropic for Claude, OpenAI for Codex). This means supporter PII in API responses is transferred to a third party during inference. We recommend using a **sandbox or test EN account** for all development work.

If you connect to a production account, read **[SECURITY.md](SECURITY.md)** first — it covers data flow, GDPR considerations, production procedures, and payment data restrictions.

## Write Safety

The `engaging-networks-api` skill protects production donor data with a three-layer safety model:

1. **Risk classification** -- every write endpoint is tagged WRITE or DESTRUCTIVE
2. **Confirmation gates** -- your agent shows you the exact request and waits for approval before any write
3. **Dry-run pattern** -- generated code includes `dryRun` option to preview without executing

See `skills/engaging-networks-api/README.md` for full details.

## License

MIT
