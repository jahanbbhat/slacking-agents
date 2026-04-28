# Slack Shortcuts Reference

> Fetched from docs.slack.dev — 2026-04-27

Shortcuts are entry points that let users invoke your app from the Slack UI without typing a slash command. Two types exist: global and message.

## Global vs Message Shortcuts

| | Global | Message |
|--|--------|---------|
| Where it appears | Shortcuts menu (⚡) in composer + search | Right-click / ⋯ menu on a message |
| Channel context | No | Yes (`shortcut.channel.id`) |
| Message context | No | Yes (`shortcut.message`, `shortcut.message_ts`) |
| `response_url` | No | Yes |
| Use when | Action doesn't need a specific message | Action operates on a specific message |
| Limit per app | 5 | 5 |
| Required scope | `commands` | `commands` |

## Global Shortcut Payload

```ts
interface GlobalShortcutPayload {
  type: 'shortcut';
  callback_id: string;       // identifies which shortcut was triggered
  trigger_id: string;        // use within 3 seconds to open a modal
  action_ts: string;
  user: { id: string; username: string; team_id: string };
  team: { id: string; domain: string };
  api_app_id: string;
  token: string;             // DEPRECATED
}
```

## Message Shortcut Payload

```ts
interface MessageShortcutPayload {
  type: 'message_action';
  callback_id: string;
  trigger_id: string;        // use within 3 seconds to open a modal
  action_ts: string;
  message_ts: string;        // ts of the message that was right-clicked
  response_url: string;      // webhook to post a message response
  message: {
    type: 'message';
    text: string;
    ts: string;
    user?: string;           // absent for bot messages
    bot_id?: string;         // present for bot messages
    blocks?: Block[];
  };
  channel: { id: string; name: string };
  user: { id: string; username: string; team_id: string };
  team: { id: string; domain: string };
  api_app_id: string;
  token: string;             // DEPRECATED
}
```

## Handling Shortcuts in Bolt

```ts
// Global shortcut
app.shortcut('create_task', async ({ shortcut, ack, client }) => {
  await ack(); // must ack within 3 seconds

  await client.views.open({
    trigger_id: shortcut.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'create_task_modal',
      title: { type: 'plain_text', text: 'Create Task' },
      submit: { type: 'plain_text', text: 'Create' },
      blocks: [ /* ... */ ],
    },
  });
});

// Message shortcut
app.shortcut('save_message', async ({ shortcut, ack, client }) => {
  await ack();

  const messageText = shortcut.message.text;
  const channelId = shortcut.channel.id;
  const messageTs = shortcut.message_ts;

  await client.views.open({
    trigger_id: shortcut.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'save_message_modal',
      title: { type: 'plain_text', text: 'Save Message' },
      submit: { type: 'plain_text', text: 'Save' },
      // Pass context through for use in view_submission handler
      private_metadata: JSON.stringify({ channelId, messageTs, messageText }),
      blocks: [
        {
          type: 'section',
          text: { type: 'mrkdwn', text: `*Message:*\n>${messageText}` },
        },
      ],
    },
  });
});
```

## Responding to a Message Shortcut Without a Modal

For message shortcuts, you can reply via `response_url`:

```ts
app.shortcut('acknowledge_message', async ({ shortcut, ack }) => {
  await ack();

  await fetch(shortcut.response_url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: `:white_check_mark: <@${shortcut.user.id}> acknowledged this message.`,
      response_type: 'in_channel',
    }),
  });
});
```

## Handling view_submission from a Shortcut-Opened Modal

```ts
app.view('save_message_modal', async ({ view, ack, client, body }) => {
  await ack();

  const { channelId, messageTs, messageText } = JSON.parse(view.private_metadata);
  const userId = body.user.id;

  // Do work with the saved context
  await saveToDatabase({ channelId, messageTs, messageText, savedBy: userId });
  await client.chat.postEphemeral({
    channel: channelId,
    user: userId,
    text: ':white_check_mark: Message saved!',
  });
});
```

## Registering Shortcuts (App Manifest)

```json
{
  "features": {
    "shortcuts": [
      {
        "name": "Create Task",
        "callback_id": "create_task",
        "description": "Create a new task from anywhere",
        "type": "global"
      },
      {
        "name": "Save Message",
        "callback_id": "save_message",
        "description": "Save this message as a task",
        "type": "message"
      }
    ]
  }
}
```

## Key Rules

- Always `ack()` within **3 seconds** — users see a generic error if you don't
- `trigger_id` from shortcuts expires in **3 seconds** — open your modal before any async work
- Global shortcuts have no channel/message context — if you need to post a message, open a modal with a conversation selector
- Message shortcuts cannot be triggered on ephemeral messages
