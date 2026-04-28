# Slack OAuth Scopes Reference

> Fetched from docs.slack.dev — 2026-04-27

## Scope Principles

- Request only the scopes your app needs — Slack reviewers reject over-permissioned apps
- Bot scopes (`scope=`) grant permissions to the bot token (`xoxb-`)
- User scopes (`user_scope=`) grant permissions to the user token (`xoxp-`)
- Scopes are additive — re-running OAuth adds scopes, never removes them
- Handle `missing_scope` errors gracefully when users deny optional scopes

```ts
// Handling missing_scope gracefully
try {
  await client.reactions.add({ channel, timestamp, name: 'thumbsup' });
} catch (error) {
  if (error.data?.error === 'missing_scope') {
    await respond({ text: 'This action requires additional permissions.', response_type: 'ephemeral' });
  }
}
```

## Bot Token Scopes (Most Common)

### Messaging

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `chat:write` | `chat.postMessage`, `chat.postEphemeral`, `chat.update`, `chat.delete`, `chat.scheduleMessage` | Core messaging scope |
| `chat:write.public` | Same as `chat:write` | Allows posting to channels the bot hasn't joined |
| `chat:write.customize` | Same as `chat:write` | Allows custom username and avatar per message |
| `incoming-webhook` | Webhook URL | Single-channel posting via webhook |

```ts
// chat:write — post a message
await client.chat.postMessage({ channel: '#general', text: 'Hello!' });

// chat:write.customize — post with custom identity
await client.chat.postMessage({
  channel: '#general',
  text: 'Hello from a custom name!',
  username: 'My Custom Bot',
  icon_emoji: ':robot_face:',
});

// chat:write.public — post without joining first
await client.chat.postMessage({ channel: 'C0CHANNEL_ID', text: 'Posting publicly!' });
```

### Reading Messages

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `channels:history` | `conversations.history`, `conversations.replies` | Read public channel messages |
| `groups:history` | `conversations.history`, `conversations.replies` | Read private channel messages |
| `im:history` | `conversations.history`, `conversations.replies` | Read DM messages |
| `mpim:history` | `conversations.history`, `conversations.replies` | Read group DM messages |

```ts
// channels:history — read recent messages
const result = await client.conversations.history({ channel: 'C0CHANNEL_ID', limit: 10 });
const messages = result.messages;
```

### Channels & Conversations

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `channels:read` | `conversations.list`, `conversations.info` | List/read public channels |
| `channels:join` | `conversations.join` | Join public channels |
| `groups:read` | `conversations.list`, `conversations.info` | List/read private channels the bot is in |
| `im:read` | `conversations.list`, `conversations.info` | List DMs |
| `im:write` | `conversations.open` | Open DMs with users |
| `mpim:read` | `conversations.list`, `conversations.info` | List group DMs |
| `mpim:write` | `conversations.open` | Open group DMs |

```ts
// im:write — open a DM and post
const dm = await client.conversations.open({ users: 'U0USER_ID' });
await client.chat.postMessage({ channel: dm.channel!.id!, text: 'Hi there!' });
```

### Users

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `users:read` | `users.info`, `users.list` | Read basic user profile info |
| `users:read.email` | `users.info` | Read user email addresses; requires `users:read` |
| `users.profile:read` | `users.profile.get` | Read detailed user profile fields |
| `users:write` | `users.setPresence` | Set bot presence |

```ts
// users:read — fetch user info
const user = await client.users.info({ user: 'U0USER_ID' });
console.log(user.user?.real_name);

// users:read.email — fetch email (requires users:read too)
const user = await client.users.info({ user: 'U0USER_ID' });
console.log(user.user?.profile?.email);
```

### App Interactions

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `app_mentions:read` | `message.app_mention` event | Receive events when bot is @mentioned |
| `commands` | N/A | Required to add slash commands |

```ts
// app_mentions:read
app.event('app_mention', async ({ event, say }) => {
  await say({ text: `Hi <@${event.user}>!`, thread_ts: event.ts });
});

// commands
app.command('/ping', async ({ ack, respond }) => {
  await ack();
  await respond('Pong!');
});
```

### Files

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `files:read` | `files.info`, `files.list` | Read file metadata and content |
| `files:write` | `files.upload`, `files.delete` | Upload and manage files |

```ts
// files:write — upload a file
await client.filesUploadV2({
  channel_id: 'C0CHANNEL_ID',
  filename: 'report.txt',
  content: 'File content here',
});
```

### Reactions

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `reactions:read` | `reactions.get`, `reactions.list`; `reaction_added` event | Read reactions |
| `reactions:write` | `reactions.add`, `reactions.remove` | Add/remove emoji reactions |

```ts
// reactions:write — add a reaction
await client.reactions.add({ channel: 'C0CHANNEL_ID', timestamp: '1234567890.123456', name: 'white_check_mark' });

// reactions:read — listen for reactions
app.event('reaction_added', async ({ event }) => {
  console.log(`${event.user} reacted with :${event.reaction}:`);
});
```

### Pins & Bookmarks

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `pins:read` | `pins.list`; `pin_added`/`pin_removed` events | Read pinned items |
| `pins:write` | `pins.add`, `pins.remove` | Pin/unpin messages |
| `bookmarks:read` | `bookmarks.list` | Read channel bookmarks |
| `bookmarks:write` | `bookmarks.add`, `bookmarks.edit`, `bookmarks.remove` | Manage bookmarks |

### Team

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `team:read` | `team.info` | Read workspace info (name, domain, icon) |

### Links / Unfurling

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `links:read` | `link_shared` event | Receive events when matching URLs are shared |
| `links:write` | `chat.unfurl` | Respond with rich unfurls |

```ts
// links — unfurl a URL
app.event('link_shared', async ({ event, client }) => {
  await client.chat.unfurl({
    channel: event.channel,
    ts: event.message_ts,
    unfurls: {
      [event.links[0].url]: {
        blocks: [{ type: 'section', text: { type: 'mrkdwn', text: '*Rich preview*' } }],
      },
    },
  });
});
```

### User Groups

| Scope | Methods Unlocked | Notes |
|-------|-----------------|-------|
| `usergroups:read` | `usergroups.list` | List user groups |
| `usergroups:write` | `usergroups.create`, `usergroups.update`, `usergroups.users.update` | Manage user groups |

### Views & Home Tab

No dedicated scope required for views. `views.open` requires a valid `trigger_id` from an interaction.

```ts
// views.publish — update App Home (no extra scope needed)
app.event('app_home_opened', async ({ event, client }) => {
  await client.views.publish({
    user_id: event.user,
    view: { type: 'home', blocks: [] },
  });
});
```

## Deprecated Scopes (Do Not Use)

| Old Scope | Replacement |
|-----------|-------------|
| `chat:write:bot` | `chat:write` |
| `chat:write:user` | `chat:write` |
| `client` | Use specific granular scopes |
| `read` | Use specific granular scopes |
| `post` | `chat:write` |

## Minimum Scope Sets for Common App Types

### Basic bot that reads mentions and replies
```
app_mentions:read, chat:write, channels:history
```

### Slash command bot
```
commands, chat:write
```

### Bot that DMs users
```
chat:write, im:write, users:read
```

### Bot that manages reactions
```
reactions:read, reactions:write, channels:history
```

### Bot with App Home tab
```
chat:write (no extra scope needed for views.publish)
```
