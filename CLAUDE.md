# slacking-agents

A collection of specialized Claude Code subagents for building Slack applications.

## How the agent system works

All feature work follows a structured flow:

```
1. slack-architect (Opus)     — produces a structured plan
2. Implementation agents      — consume the plan and write code
3. slack-reviewer (Sonnet)    — checks the produced code and reports findings
```

Always invoke `slack-architect` first. Implementation agents do not make design decisions — if they encounter ambiguity in a plan, they surface the options back to `slack-architect` for a decision.

## Design principles

These principles inform how the agent system is structured:

- **TypeScript only.** All generated code and reference docs use TypeScript. Reason: the project targets the Bolt.js ecosystem.
- **Opus plans, Sonnet/Haiku implements.** `slack-architect` (Opus) produces plans; implementation agents (Sonnet/Haiku) consume them. Reason: keeps Opus invocations rare and deliberate; cheaper models handle mechanical work.
- **Implementation agents escalate ambiguity.** Implementation agents do not make design decisions — they surface options and tradeoffs back to the architect. Reason: keeps design authority with the planner; prevents implementation agents from making security-sensitive judgment calls.

## Agents

### Orchestrator

| Agent | Model | Purpose |
|-------|-------|---------|
| `slack-orchestrator` | Sonnet | Primary entry point — invokes architect then sequences implementation agents automatically |

### Planner

| Agent | Model | Purpose |
|-------|-------|---------|
| `slack-architect` | Opus | Designs features, produces implementation plans, resolves ambiguity |

### Implementation

| Agent | Model | Purpose |
|-------|-------|---------|
| `slack-bootstrap-agent` | Sonnet | Project scaffolding (package.json, tsconfig, app.ts, .env.example) — only invoked for greenfield projects |
| `slack-oauth-agent` | Sonnet | OAuth flows, token storage, signing secret verification |
| `block-kit-builder` | Sonnet | Block Kit JSON, modals, view submissions |
| `slack-event-router` | Sonnet | Event subscriptions, middleware, URL verification |
| `slash-command-handler` | Sonnet | Slash command endpoints, deferred responses |
| `slack-rate-limit-handler` | Sonnet | Retry logic, queuing, backoff |
| `slack-message-formatter` | Haiku | mrkdwn text, threading, DMs, ephemeral messages |
| `slack-payload-mocker` | Haiku | Mock payloads for unit and integration tests |
| `slack-reviewer` | Sonnet | Report-only quality gate; invoked automatically by orchestrator as the final step |

## Typical workflow

For most features, invoke `slack-orchestrator` directly with a description of what you want to build. It will invoke `slack-architect` to produce a plan, resolve any Open Questions and `[VERIFY]` tags with you, then sequence the implementation agents automatically.

```
User → slack-orchestrator → slack-architect (plan) → [Open Questions / VERIFY gates] → [bootstrap?] → implementation agents → slack-reviewer → code
```

For debugging or targeted work on a specific layer, invoke the relevant implementation agent directly with a copy of the relevant plan section.

**Manual example** (without orchestrator):
```
User: "Build a /standup slash command that opens a modal and posts to #standups"

1. Invoke slack-architect
   → Produces plan: surfaces, data flow, types, edge cases, agent assignments

2. slack-bootstrap-agent scaffolds the project (if greenfield)
3. slash-command-handler builds the /standup endpoint (parallel with block-kit-builder)
4. block-kit-builder builds the modal UI (parallel with slash-command-handler)
5. slack-event-router wires any needed event listeners
6. slack-message-formatter handles the post to #standups
7. slack-payload-mocker generates test fixtures
8. slack-reviewer checks all produced code and reports Critical / Should fix / Nit findings
```

## Project structure

All implementation agents write to a standard directory layout:

```
src/
  app.ts                 — bootstrap, App constructor, app.start() (slack-bootstrap-agent)
  oauth/                 — installation flows, InstallationStore (slack-oauth-agent)
  events/                — app.event() listeners (slack-event-router)
  commands/              — app.command() handlers (slash-command-handler)
  ui/
    blocks/              — Block Kit definitions, modals, home tabs (block-kit-builder)
    messages/            — mrkdwn message templates and posting helpers (slack-message-formatter)
  lib/
    rate-limit.ts        — retry/queue utilities (slack-rate-limit-handler)
    constants.ts         — exported callback_ids, action_ids, route paths
tests/
  fixtures/              — mock Slack payloads (slack-payload-mocker)
artifacts/
  plans/<name>.md        — architect plans (written by orchestrator after planning)
  reviews/<name>.md      — reviewer reports (written by slack-reviewer)
  build-state/<name>.md  — orchestrator build state checkpoints (written incrementally)
```

## Local reference docs

All agents read from `docs/slack/` before relying on training knowledge. Each agent's prompt lists the specific files relevant to its scope. Do not delete or rename these files.

| File | Contents |
|------|----------|
| `oauth-v2.md` | OAuth flow, signing secret verification, token storage |
| `scopes.md` | All OAuth scopes with TS examples |
| `block-kit-blocks.md` | Block types (section, actions, input, header…) |
| `block-kit-elements.md` | Interactive elements + multi-selects |
| `composition-objects.md` | text, option, confirm, filter, dispatch_action_config objects |
| `views-modals.md` | views.open/push/update/publish, all payload shapes |
| `mrkdwn-formatting.md` | Full mrkdwn syntax reference |
| `block-actions.md` | block_actions payload, reading values, response patterns |
| `slash-commands.md` | Payload shape, response_url, deferred responses |
| `shortcuts.md` | Global vs message shortcuts, Bolt examples |
| `incoming-webhooks.md` | Webhook setup, payload format, limitations |
| `file-upload.md` | 3-step upload flow |
| `app-manifest.md` | Full manifest schema |
| `bolt-initialization.md` | App constructor, Socket Mode vs HTTP, multi-workspace setup |
| `bolt-types.md` | Bolt TypeScript types: SlashCommand, BlockAction, SayFn, etc. |
| `error-codes.md` | All Slack API error strings with recovery actions |
| `pagination.md` | Cursor-based pagination for list methods |
| `events.md` | All 120+ event types with required scopes |
| `rate-limits.md` | Tier table, method assignments, retry rules |
| `dm-conversations.md` | Opening DMs, `message.im` listener, multi-turn conversation state, fetching channel members |

## Refreshing docs

The `docs/slack/` files were last fetched **2026-04-27**. Refresh them when:
- Slack announces a new API feature or deprecation (check [api.slack.com/changelog](https://api.slack.com/changelog))
- An agent produces code that hits unexpected `invalid_arguments` errors
- You add a new agent that needs API surface not covered by existing docs

**How to refresh:**
1. Open `.claude/agents/slack-architect.md`
2. Uncomment the `# - WebFetch` line in the tools list
3. Ask `slack-architect` to re-fetch and rewrite the relevant doc
4. Re-comment the `WebFetch` line when done

## WebFetch

`slack-architect` has `WebFetch` commented out in its tools list. Uncomment it when live Slack API doc lookups are needed (e.g. verifying a new API feature or checking for deprecation notices). Re-comment when done to keep the agent operating from local docs by default.

## Adding new agents

1. Create `.claude/agents/<name>.md` with frontmatter (`name`, `description`, `model`, `tools`)
2. Add the agent to the Implementation Agent Assignments section in `slack-architect.md`
3. Update this file's agent table

## Code style

All code examples and generated code should be TypeScript.
