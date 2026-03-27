# Contributing

Contributions to the Orkestre Engaging Networks skills are welcome. This guide covers how to contribute effectively.

## Ways to Contribute

- **Bug reports** -- Skill gave incorrect endpoint paths, wrong auth flow, or bad code patterns
- **Endpoint corrections** -- EN updated their API and something is out of date
- **New workflows** -- Additional guided workflows for common integration tasks
- **Reference improvements** -- Better examples, additional common patterns, new language support
- **Documentation fixes** -- Typos, unclear instructions, missing prerequisites

## Before You Start

**For non-trivial changes**, open an issue first to discuss the approach. This prevents wasted effort on changes that may not align with the project's direction.

**EN API access is required** to meaningfully test changes. You'll need an API User token and IP whitelisting configured in your EN admin panel. See the [README](README.md#prerequisites) for setup details.

## Submitting Changes

1. Fork the repository
2. Create a branch from `main`
3. Make your changes
4. Test against a live EN API account if your changes affect endpoint references or workflows
5. Submit a pull request with a clear description of what changed and why

## Guidelines

### Write Safety is Non-Negotiable

The skill protects production nonprofit donor data. PRs that do any of the following require explicit justification and careful review:

- Change an endpoint's risk classification (READ/WRITE/DESTRUCTIVE)
- Remove or weaken confirmation gates for write operations
- Remove the dry-run pattern from generated client code

### Preserve Skill Structure

The skill uses progressive disclosure to minimize token usage. Maintain these conventions:

- **SKILL.md** stays under 200 lines (intake, routing, safety rules only)
- **References** contain detailed information loaded on demand
- **Workflows** are self-contained step-by-step guides
- YAML frontmatter in SKILL.md must have accurate `name` and `description` fields

### No Secrets in PRs

Never include real API tokens, supporter IDs, or client-specific data in commits. Use placeholder values in examples (e.g., `YOUR_API_USER_TOKEN`, `supporter-id-here`).

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
