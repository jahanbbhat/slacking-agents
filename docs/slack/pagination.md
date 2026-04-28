# Slack API Pagination Reference

> Fetched from docs.slack.dev — 2026-04-27

Most Slack list methods use **cursor-based pagination**. Never assume a single call returns all results.

## How It Works

1. Make initial request (no `cursor`)
2. Check `response_metadata.next_cursor` in the response
3. If non-empty, make another request with `cursor=<next_cursor>`
4. Stop when `next_cursor` is empty or absent

## Key Fields

| Field | Location | Description |
|-------|----------|-------------|
| `limit` | Request param | Max results per page; recommended 100–200; max 1000 |
| `cursor` | Request param | Opaque pointer from previous response |
| `has_more` | Response body | `true` if more results exist (not always present) |
| `response_metadata.next_cursor` | Response body | Cursor for the next page; empty string when done |

> Always check `next_cursor` — you may receive fewer items than `limit` even when more exist.

## Methods That Support Pagination

| Method | Result field | Notes |
|--------|-------------|-------|
| `conversations.list` | `channels` | Tier 2 — paginate carefully |
| `conversations.history` | `messages` | Tier 3 |
| `conversations.replies` | `messages` | Tier 3 |
| `conversations.members` | `members` | Tier 4 |
| `users.list` | `members` | Tier 2 |
| `reactions.list` | `items` | Tier 2 |
| `files.list` | `files` | Tier 3 |
| `stars.list` | `items` | Tier 2 (deprecated in favor of bookmarks) |

## TypeScript Patterns

### Generic paginator using the Web API client

```ts
import { WebClient } from '@slack/web-api';

async function paginateAll<T>(
  client: WebClient,
  method: string,
  params: Record<string, unknown>,
  resultKey: string,
): Promise<T[]> {
  const results: T[] = [];
  let cursor: string | undefined;

  do {
    const response = await (client as any)[method]({
      ...params,
      limit: 200,
      ...(cursor ? { cursor } : {}),
    });

    results.push(...(response[resultKey] ?? []));
    cursor = response.response_metadata?.next_cursor || undefined;
  } while (cursor);

  return results;
}

// Usage
const allChannels = await paginateAll(client, 'conversations.list', { types: 'public_channel' }, 'channels');
const allUsers    = await paginateAll(client, 'users.list', {}, 'members');
```

### Using the built-in async iterator (recommended)

The `@slack/web-api` client exposes `paginate()` which handles cursor iteration automatically:

```ts
import { WebClient } from '@slack/web-api';

const client = new WebClient(process.env.SLACK_BOT_TOKEN);

// Collect all public channels
const channels: any[] = [];
for await (const page of client.paginate('conversations.list', { types: 'public_channel', limit: 200 })) {
  channels.push(...(page.channels ?? []));
}

// Collect all workspace members
const users: any[] = [];
for await (const page of client.paginate('users.list', { limit: 200 })) {
  users.push(...(page.members ?? []));
}

// Collect all messages in a channel
const messages: any[] = [];
for await (const page of client.paginate('conversations.history', { channel: 'C0CHANNEL_ID', limit: 200 })) {
  messages.push(...(page.messages ?? []));
}
```

### Manual cursor loop (when you need early exit)

```ts
async function findUserByEmail(client: WebClient, email: string) {
  let cursor: string | undefined;

  do {
    const response = await client.users.list({ limit: 200, cursor });
    const match = response.members?.find(u => u.profile?.email === email);
    if (match) return match;
    cursor = response.response_metadata?.next_cursor || undefined;
  } while (cursor);

  return null;
}
```

## Rate Limit Considerations

Paginating through large workspaces can burn your rate limit quickly. Follow these rules:

```ts
// Tier 2 methods (conversations.list, users.list) allow ~20 req/min
// At 200 results/page, a 10,000-member workspace needs 50 pages
// 50 pages at Tier 2 = ~2.5 minutes minimum — plan accordingly

// Add delay between pages for Tier 2 methods on large workspaces
async function paginateWithDelay<T>(
  client: WebClient,
  method: string,
  params: Record<string, unknown>,
  resultKey: string,
  delayMs = 1000,
): Promise<T[]> {
  const results: T[] = [];
  let cursor: string | undefined;
  let page = 0;

  do {
    if (page > 0) await new Promise(r => setTimeout(r, delayMs));
    const response = await (client as any)[method]({ ...params, limit: 200, cursor });
    results.push(...(response[resultKey] ?? []));
    cursor = response.response_metadata?.next_cursor || undefined;
    page++;
  } while (cursor);

  return results;
}
```

## Cursor Expiration and Encoding

- Cursors expire — use them promptly within a single paginated session
- Cursor strings ending with `=` must be URL-encoded as `%3D` when passed as query params
- The `@slack/web-api` client handles encoding automatically — only manually encode if constructing raw HTTP requests
- `invalid_cursor` error means the cursor is malformed, incorrectly encoded, or expired

## Common Mistakes

```ts
// ❌ Wrong — assumes all results fit in one call
const { channels } = await client.conversations.list();

// ✓ Correct — paginate to get all results
const channels = [];
for await (const page of client.paginate('conversations.list', { limit: 200 })) {
  channels.push(...(page.channels ?? []));
}

// ❌ Wrong — checking has_more instead of next_cursor
while (response.has_more) { ... }

// ✓ Correct — always check next_cursor
while (response.response_metadata?.next_cursor) { ... }
```
