# Slack API Rate Limits Reference

> Fetched from docs.slack.dev — 2026-04-27

## Tier Overview

| Tier | Limit | Notes |
|------|-------|-------|
| Tier 1 | 1+ req/min | Infrequent methods; minor burst tolerance |
| Tier 2 | 20+ req/min | Most methods; occasional bursts permitted |
| Tier 3 | 50+ req/min | Larger quotas; paginated collection methods |
| Tier 4 | 100+ req/min | High-volume; most generous burst behavior |
| Special | Varies | Method-specific conditions |

## Common Method Tiers

| Method | Tier |
|--------|------|
| `chat.postMessage` | Tier 3 (+ 1 msg/sec per channel) |
| `chat.postEphemeral` | Tier 4 |
| `chat.update` | Tier 3 |
| `chat.delete` | Tier 3 |
| `conversations.list` | Tier 2 (Special — lower than tier suggests) |
| `conversations.history` | Tier 3 |
| `conversations.replies` | Tier 3 |
| `conversations.open` | Tier 3 |
| `users.info` | Tier 4 |
| `users.list` | Tier 2 |
| `reactions.add` | Tier 3 |
| `views.open` | Tier 4 |
| `views.update` | Tier 4 |
| `views.publish` | Tier 4 |

## Special Limits

- **`chat.postMessage`**: Additionally capped at 1 message/second per channel (applies to incoming webhooks and RTM too)
- **`users.profile.set`**: Max 10 updates/min per user, 30 profiles/min per token
- **Events API delivery**: 30,000 deliveries per workspace per app per 60 minutes
- **`conversations.history` / `conversations.replies`**: Non-Marketplace apps face additional restrictions (as of May 29, 2025); Marketplace and internal apps unaffected

## Rate Limit Responses

When exceeded:
- HTTP status: `429 Too Many Requests`
- Header: `Retry-After: <seconds>` — always use this value, never guess

## Implementation Rules

1. Always read `Retry-After` from the 429 response — do not guess or use fixed intervals
2. Use exponential backoff with jitter between retries
3. Design around 1 request/second as a sustainable baseline for `chat.postMessage`
4. For bulk operations (e.g., messaging many users), use a queue with per-method concurrency limits
5. Cap retry attempts (recommended max: 3) and surface a typed error after exhaustion
6. Never silently swallow `ratelimited` errors — log method name, retry count, and wait time

## Burst Behavior

Slack permits temporary burst capacity beyond stated limits but does not publish exact burst thresholds — they change without notice. Do not design systems that rely on burst capacity.
