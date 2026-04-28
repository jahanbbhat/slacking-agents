# Async Standup Bot — Product & Feature Spec

**Version:** 1.0  
**Scope:** Single Slack workspace, MVP core feature set  
**Inspired by:** Standuply

---

## Overview

An async standup bot that eliminates the need for synchronous daily standup meetings. Any team member adds the bot to a channel, which becomes the standup's home. From there, any team member can trigger a cycle via slash command — the bot privately DMs each participant a set of standup questions and compiles responses into a report posted back to that channel.

---

## Core User Flows

### 1. Channel Activation Flow

Any workspace member adds the bot to a channel using Slack's native "Add apps" UX or `/invite @standup-bot`. On joining, the bot posts a welcome message in that channel:

> _"Hi! I'm your standup bot. Run `/standup setup` to configure participants and questions, then `/standup run` when you're ready to kick off a cycle. Use `/standup status` to see the current config."_

The channel the bot is added to is the channel the standup serves — reports post there and commands are scoped to it. If a command is invoked from a channel the bot is not a member of, the bot responds ephemerally: _"Add me to a channel first with `/invite @standup-bot`, then run this command from there."_

### 2. Setup Flow

Any team member configures the standup via a `/standup setup` slash command. This opens a modal where they set:

- **Participants** — select users or pull all members from a specific channel
- **Questions** — start from the default template; add, remove, or reorder questions
- **Response window** — how long participants have to respond after a cycle is triggered (e.g. 2 hours)

Configuration is saved per-workspace. Only one standup configuration exists in V1.

### 3. Standup Collection Flow

When `/standup run` is issued, the bot:

1. DMs each participant individually with the first question
2. Waits for their reply, then sends the next question
3. Repeats until all questions are answered
4. Confirms submission: _"Thanks! Your standup has been recorded."_

The conversation is conversational and sequential — one question at a time, not a form. This keeps the UX lightweight and works well on mobile.

### 4. Reminder Flow

If a participant has not responded within a configurable reminder window (default: 30 minutes before the response window closes), the bot sends a single follow-up DM nudge:

> _"Reminder: your standup closes in 30 minutes. Reply here to submit."_

Only one reminder per standup cycle. No repeated pinging.

### 5. Report Compilation Flow

When the response window closes, the bot:

1. Collects all submitted responses
2. Compiles them into a structured Block Kit message
3. Posts the report to the configured report channel
4. Lists non-responders at the bottom of the report (no shaming — just visibility)

Reports are posted once per cycle, at deadline time. There is no "async delivery as responses come in" in V1.

---

## Features

### Slash Commands

| Command | Description |
|---------|-------------|
| `/standup setup` | Opens setup modal |
| `/standup run` | Triggers a standup cycle immediately |
| `/standup status` | Shows current config and whether a cycle is active |

Commands invoked from a channel the bot is not a member of return an ephemeral error pointing the user to `/invite @standup-bot` first.

### Default Standup Questions

The bot ships with these three questions pre-configured (editable):

1. What did you accomplish yesterday?
2. What are you working on today?
3. Any blockers or impediments?

### Report Format

The compiled report posted to the channel is structured as:

```
📋 Daily Standup — [Date]
──────────────────────────
@alice
  Yesterday: Finished the auth refactor
  Today: Starting on rate limiting
  Blockers: None

@bob
  Yesterday: Reviewed PRs
  Today: Writing tests for payment flow
  Blockers: Waiting on API keys from DevOps

──────────────────────────
✅ Responded: @alice, @bob (2/3)
⏳ No response: @charlie
```

Blockers that are not "None" / "nothing" / empty are visually highlighted (bold or an emoji prefix) to draw attention.

### Participant Management

- Any team member can add/remove participants at any time via `/standup setup`
- Adding a Slack channel auto-enrolls all current members of that channel
- Participants who are deactivated or leave the workspace are automatically skipped

---

## Out of Scope for V1

These Standuply features are intentionally excluded from V1:

- Scheduled/automatic standup triggering (V1 is slash-command triggered only)
- Multiple standup configurations
- Video/voice responses
- Jira, GitHub, or other integrations
- AI-generated summary of the report
- Analytics or historical reporting
- Email delivery of reports
- Multi-workspace support

---

## Key Design Decisions

**Sequential DM questions, not a modal form.** Modals require the user to be active in Slack at the exact moment the standup fires. Sequential DMs allow responses to trickle in asynchronously over the response window.

**Channel-scoped activation.** The bot must be added to a channel before it can be used. The channel it joins becomes the implicit report channel — no separate config field needed. This follows standard Slack app UX (opt-in per channel) and eliminates a configuration step from the setup modal.

**No role gates.** Any workspace member can add the bot, run setup, or trigger a cycle. This keeps the bot frictionless for small teams where everyone shares ownership of the standup.

**Slash-command triggered, not scheduled.** V1 requires a team member to run `/standup run` to kick off a cycle. This eliminates the need for a persistent scheduler and simplifies the implementation significantly. Automatic scheduling is a natural V2 addition once the core flow is proven.

**One reminder max.** Standuply supports persistent reminders; this bot sends exactly one to avoid notification fatigue.

**No async drip posting.** V1 posts the full report once at deadline. Drip posting (posting each answer as it comes in) adds complexity and can create noise in the channel before everyone has responded.

---

## Technical Constraints (for the implementing engineer)

- Built on **Bolt.js (TypeScript)** targeting a single Slack workspace
- Config (participants, questions, response window) must persist across restarts — use a simple store (e.g. a JSON file or SQLite); the home channel ID is part of this config
- Active cycle state (who has responded, their answers, when the window closes) also needs to persist across restarts so an in-progress cycle survives a server bounce
- Bot subscribes to the `member_joined_channel` event and filters to its own user ID to detect when it's been added to a channel
- Bot user ID is fetched at startup via `auth.test` and cached in memory for the lifetime of the process
- All Slack interactions use Block Kit for the report and plain Slack modals for setup
- The bot needs the following OAuth scopes: `chat:write`, `im:write`, `commands`, `users:read`, `channels:read`
- No OAuth install flow needed (single workspace — use a bot token directly from environment variables)
