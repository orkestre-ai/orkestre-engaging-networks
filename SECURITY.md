# Security & Data Privacy

This document explains how data flows when you use an AI coding agent with the Engaging Networks API, what that means for your organization, and how to use this skill responsibly.

## What This Skill Is For

This skill is a **development assistant**. It helps you write, debug, optimize, and deploy code that integrates with the Engaging Networks REST API. Its full value — endpoint references, code generation, authentication patterns, deployment guidance — is available **without real donor data**.

It is not designed to be a live data operations proxy against production accounts.

## How Data Flows

When your AI agent makes an EN API call, the response travels through multiple systems:

```
Engaging Networks servers
        |
        v
Your local machine (CLI session)
        |
        v
Model provider infrastructure (Anthropic, OpenAI, etc.)
        |
        v
Back to your local machine (agent response)
```

Every API response that contains supporter data — names, emails, addresses, donation history — is processed by the model provider's infrastructure during inference. This is how the technology works; the agent cannot reason about data without its provider processing that data.

**This constitutes a data transfer to a third party.** Your supporters did not consent to this processing, and most nonprofit data processing agreements do not account for it.

## Recommended: Use a Sandbox Account

All skill workflows work against sandbox or test EN accounts. We recommend this as the default:

- **Building an integration** — sandbox provides the same API surface
- **Debugging API issues** — schema endpoints (`/supporter/fields`, `/page?type=dc`) expose no PII
- **Generating client code** — requires endpoint knowledge, not real data
- **Testing** — test accounts exist for this purpose

Your EN account manager can provision a sandbox environment if you don't have one.

## If You Use a Production Account

If you deliberately connect to a production EN account through the agent, understand what that means:

### Before you start

- **Inform your Data Protection Officer** (if your organization has one) that donor data will be processed by a third-party AI model provider
- **Check your Data Processing Agreements** — you likely need a DPA with the model provider (e.g., [Anthropic's DPA](https://www.anthropic.com/policies), [OpenAI's DPA](https://openai.com/policies))
- **Confirm with your EN account administrator** that API use through an AI agent is compatible with your organization's data handling policies

### While working

- **Query only what you need.** Use `/supporter/fields` (no PII) before `/supporter?email=...` (PII). Use targeted lookups, not bulk queries.
- **Avoid bulk data retrieval through the agent.** If you need to pull large datasets, use EN's Export Jobs feature in your application code, not through the agent session.
- **Review API responses before proceeding.** When the agent shows you data, verify it pulled only what was necessary.

### After your session

- **Clear your conversation history** if it contains donor PII. AI coding tools store conversation logs locally and may retain them on provider infrastructure.
- **Retire your session token** — the agent-generated code includes token cleanup, but verify the session was closed.

## Payment Data — Do Not Use Through Agent

The EN API includes a `-vault` host variant for processing credit card payments (e.g., `ca-vault.engagingnetworks.app`). **Never route payment card data through an AI agent session.**

Credit card numbers, CVVs, and expiration dates in an agent's context window create PCI DSS compliance exposure. No approval gate or confirmation prompt mitigates this. Process payments through direct server-to-EN connections in your application code, never through the agent.

## GDPR Considerations

Organizations on the EU region (`eu.engagingnetworks.app`) or processing EU residents' data should assess:

- **Lawful basis**: What is your legal basis for transferring supporter PII to an AI model provider? The original consent for data collection likely does not cover this.
- **Cross-border transfer**: Anthropic and OpenAI are US-based. Transferring EU supporter data to US infrastructure requires appropriate safeguards (Standard Contractual Clauses or equivalent).
- **Right to erasure (Art. 17)**: If a supporter requests deletion, can you account for data that passed through agent sessions? Conversation logs may retain PII beyond your control.
- **Data Protection Impact Assessment**: Depending on the volume and sensitivity of data processed, a DPIA may be warranted before using AI agents with production donor data.

These considerations apply to any AI-mediated access to personal data, not just this skill.

## What the Skill Already Protects

This skill includes safeguards for data integrity (preventing unintended modifications):

1. **Risk classification** — every endpoint is tagged READ, WRITE, or DESTRUCTIVE
2. **Confirmation gates** — the agent shows you the exact request and waits for approval before any write or delete
3. **Dry-run pattern** — generated client code includes a `dryRun` option to preview write operations without executing them
4. **Token security** — API tokens stored in environment variables, IP whitelisting required, 90-day rotation recommended

These protect against **accidental data modification**. They do not address **data privacy during read operations**, which is the primary concern this document covers.

## Model Provider Data Policies

At time of writing:

- **Anthropic (Claude)**: Does not use API or Claude Code conversations for model training. Retains conversation data for safety monitoring and abuse prevention. See their [privacy policy](https://www.anthropic.com/privacy) and [data processing terms](https://www.anthropic.com/policies).
- **OpenAI (Codex)**: Data handling varies by product and API tier. See their [enterprise privacy](https://openai.com/enterprise-privacy) documentation.

**These policies can change.** Your organization's compliance posture should not depend solely on a provider's current terms. A Data Processing Agreement provides contractual protection.

## Local Model Alternative

If your organization requires that donor data never leave your infrastructure, you can run this skill with a **locally-hosted model** instead of a cloud provider. When the model runs on your machine, API response data stays on-premise — no third-party data transfer occurs.

### How it works

This skill follows the [Agent Skills specification](https://agentskills.io/specification), which is supported by multiple agent frameworks. To use a local model:

1. **Install a local model runtime** like [Ollama](https://ollama.com)
2. **Pull a model with tool-calling support** — we recommend Qwen 2.5 Coder 32B or Llama 3.3 70B for the best balance of code quality and skill comprehension
3. **Use an agent framework that supports both local models and the Agent Skills spec** — currently Codex CLI can read `skills/` directories and be configured to use OpenAI-compatible local endpoints

```bash
# Example: Ollama setup
brew install ollama
ollama pull qwen2.5-coder:32b
```

### Trade-offs

- **Code generation quality will be lower** than cloud models (Claude, GPT-4). Complex multi-step workflows may be partially followed.
- **Hardware requirements are significant.** A 32B model needs 24GB+ VRAM; a 70B model needs 48GB+ or will run slowly on CPU.
- **This configuration has not been formally tested with this skill.** The skill was verified against Claude and Codex with cloud models.

### Hybrid approach

For most organizations, a practical middle ground:

| Task | Model | Rationale |
|------|-------|-----------|
| Building integrations, learning endpoints | Cloud model (sandbox account) | Maximum capability, no real PII involved |
| Debugging or querying production data | Local model | PII stays on your infrastructure |
| Bulk data operations | No agent — application code | Run your server-side code directly |
| Payment processing | No agent — direct server-to-EN | PCI scope, no exceptions |

This gives you cloud-model quality where it's safe (sandbox, documentation) and local-model privacy where it matters (production data).

## Summary

| Use Case | Risk Level | Recommendation |
|----------|-----------|----------------|
| Building/debugging integrations with sandbox | Low | Default mode. No real PII involved. |
| Generating client code, learning endpoints | None | No API connection needed — skill uses documentation. |
| Querying production data (cloud model) | High | Follow production procedures above. Inform DPO. |
| Querying production data (local model) | Low | PII stays on-premise. No third-party transfer. |
| Bulk data operations through agent | Very High | Don't. Use Export Jobs in your application code. |
| Payment processing through agent | Unacceptable | Never. Direct server-to-EN only. |

## Questions

If you have questions about data handling in this skill, open an issue at [orkestre-ai/orkestre-engaging-networks](https://github.com/orkestre-ai/orkestre-engaging-networks/issues).
