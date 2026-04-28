# DM Conversations in Bolt.js

Covers opening direct messages, sending bot-initiated DMs, listening for user replies in DMs, and managing per-user conversation state for multi-turn flows.

---

## Opening a DM Channel

Slack DMs have a channel ID just like public channels. Before posting to a user's DM, you must resolve their user ID to a DM channel ID using `conversations.open`.

```typescript
import { WebClient } from '@slack/web-api';

async function openDM(client: WebClient, userId: string): Promise<string> {
  const result = await client.conversations.open({ users: userId });
  return result.channel!.id!;
}
```

Required scope: `im:write`

Cache the returned channel ID — you do not need to call `conversations.open` again for the same user within the same session.

---

## Sending a DM

Once you have the DM channel ID, use `chat.postMessage` as normal:

```typescript
await client.chat.postMessage({
  channel: dmChannelId,
  text: 'What did you accomplish yesterday?',
});
```

Required scope: `chat:write`

---

## Listening for DM Replies

Use `app.message()` with a `channel_type` filter set to `'im'` to receive only direct messages. Without this filter, the listener fires on all messages in all channel types the bot can see.

```typescript
app.message(async ({ message, client }) => {
  // message.channel_type === 'im' is guaranteed by the filter below
  if (message.channel_type !== 'im') return;

  const userId = message.user;
  const text = 'text' in message ? message.text ?? '' : '';

  // route to your conversation state handler
  await handleDMReply(userId, text, client);
});
```

Required scope: `im:history`  
Required event subscription: `message.im`

**Important:** Bolt will also fire this listener for messages the bot itself sends in DMs. Guard against this by checking `message.bot_id`:

```typescript
app.message(async ({ message, client }) => {
  if (message.channel_type !== 'im') return;
  if ('bot_id' in message && message.bot_id) return; // ignore own messages

  const userId = message.user;
  // ...
});
```

---

## Multi-Turn Conversation State

For sequential Q&A (ask Q1 → wait for reply → ask Q2 → ...), you need per-user state that tracks:

- Which question the user is currently on
- Their answers so far
- The cycle they belong to (if multiple cycles could overlap)

### In-Memory State (simple, no persistence)

```typescript
interface UserConversationState {
  cycleId: string;
  questionIndex: number;
  answers: string[];
}

const activeConversations = new Map<string, UserConversationState>();
// key: userId
```

**Limitation:** state is lost on server restart. For a bot that runs production cycles, persist this to a store (see below).

### Persistent State Pattern

Use the same store you use for config (JSON file or SQLite). Write after every answer so a restart mid-cycle can resume:

```typescript
interface CycleState {
  cycleId: string;
  startedAt: string;      // ISO timestamp
  windowClosesAt: string; // ISO timestamp
  participants: {
    [userId: string]: {
      dmChannelId: string;
      questionIndex: number;
      answers: string[];
      completed: boolean;
      reminded: boolean;
    };
  };
}
```

Read this state on startup. If `windowClosesAt` is in the future and `completed` is false for some users, the cycle is still active — resume listening for their replies.

---

## Full Sequential Q&A Flow

```typescript
const QUESTIONS = [
  'What did you accomplish yesterday?',
  'What are you working on today?',
  'Any blockers or impediments?',
];

async function startConversation(
  userId: string,
  dmChannelId: string,
  client: WebClient,
  cycleId: string
) {
  activeConversations.set(userId, { cycleId, questionIndex: 0, answers: [] });
  await client.chat.postMessage({
    channel: dmChannelId,
    text: QUESTIONS[0],
  });
}

async function handleDMReply(
  userId: string,
  text: string,
  client: WebClient
) {
  const state = activeConversations.get(userId);
  if (!state) return; // no active standup for this user

  state.answers.push(text);
  state.questionIndex += 1;

  if (state.questionIndex < QUESTIONS.length) {
    const dmChannelId = await openDM(client, userId);
    await client.chat.postMessage({
      channel: dmChannelId,
      text: QUESTIONS[state.questionIndex],
    });
  } else {
    activeConversations.delete(userId);
    const dmChannelId = await openDM(client, userId);
    await client.chat.postMessage({
      channel: dmChannelId,
      text: "Thanks! Your standup has been recorded.",
    });
    // persist answers to cycle store and check if all participants are done
  }
}
```

---

## Fetching Channel Members

To pre-populate participants from a channel, use `conversations.members` with cursor-based pagination:

```typescript
async function getChannelMembers(
  client: WebClient,
  channelId: string
): Promise<string[]> {
  const members: string[] = [];
  let cursor: string | undefined;

  do {
    const result = await client.conversations.members({
      channel: channelId,
      limit: 200,
      cursor,
    });
    members.push(...(result.members ?? []));
    cursor = result.response_metadata?.next_cursor;
  } while (cursor);

  return members;
}
```

Required scope: `channels:read` (public channels) or `groups:read` (private channels)

Filter out bot users after fetching — check `users.info` for each member and skip those with `is_bot: true`, or maintain a known bot user ID list.

---

## Required Scopes Summary

| Scope | Used for |
|-------|----------|
| `im:write` | Opening DM channels via `conversations.open` |
| `im:history` | Receiving `message.im` events |
| `chat:write` | Sending messages to DM channels |
| `channels:read` | Fetching members of public channels |
| `users:read` | Filtering bot users from participant lists |

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `channel_not_found` | DM channel ID not yet opened | Call `conversations.open` first |
| `not_in_channel` | Bot not in the channel you're trying to read members from | Invite the bot or use `conversations.join` |
| `missing_scope` | Scope not granted | Add scope to app manifest and reinstall |
