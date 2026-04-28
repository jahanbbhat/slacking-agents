# Slack Slash Commands Reference

> Fetched from docs.slack.dev — 2026-04-27

## Payload Shape

When a slash command is invoked, Slack sends an HTTP POST to your request URL with this body:

```ts
interface SlashCommandPayload {
  command: string;          // e.g. "/weather"
  text: string;             // everything the user typed after the command
  user_id: string;          // always use this, not user_name
  user_name: string;        // DEPRECATED — unreliable, do not use
  channel_id: string;       // always use this, not channel_name
  channel_name: string;     // unreliable — "directmessage" or "privategroup" for non-public
  team_id: string;
  team_domain: string;
  enterprise_id?: string;   // present on Enterprise Grid installs
  enterprise_name?: string;
  response_url: string;     // webhook for deferred responses (valid 30 min, max 5 uses)
  trigger_id: string;       // use within 3 seconds to open a modal
  api_app_id: string;
  token: string;            // DEPRECATED — use signing secret verification instead
}
```

> Always use `user_id` and `channel_id` for lookups — never `user_name` or `channel_name`.

## Response Requirements

You must acknowledge with HTTP 200 within **3 seconds**.

```ts
// Bolt handles ack() for you
app.command('/weather', async ({ command, ack, respond, client }) => {
  // Acknowledge immediately
  await ack();

  // Do async work, then respond via respond() or response_url
  const forecast = await fetchForecast(command.text);
  await respond({ text: forecast });
});
```

## Response Visibility

```ts
// Ephemeral — only the user who ran the command sees it (default)
await ack({ response_type: 'ephemeral', text: 'Processing...' });

// In-channel — visible to everyone in the channel
await respond({ response_type: 'in_channel', text: 'Here is the weather.' });
```

## Deferred Responses via response_url

Use `response_url` when work takes longer than 3 seconds. Valid for **30 minutes**, **max 5 uses**.

```ts
app.command('/report', async ({ command, ack }) => {
  await ack({ response_type: 'ephemeral', text: 'Generating report...' });

  // Do slow work
  const report = await generateReport();

  // Post result via response_url
  await fetch(command.response_url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      response_type: 'in_channel',
      text: report,
    }),
  });
});
```

### response_url message operations

```ts
// Replace the original message
await fetch(command.response_url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ replace_original: true, text: 'Updated!' }),
});

// Delete the original message
await fetch(command.response_url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ delete_original: true }),
});
```

## Opening a Modal from a Slash Command

Use `trigger_id` within **3 seconds** of receiving the command.

```ts
app.command('/create', async ({ command, ack, client }) => {
  await ack();

  await client.views.open({
    trigger_id: command.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'create_modal',
      title: { type: 'plain_text', text: 'Create Item' },
      submit: { type: 'plain_text', text: 'Submit' },
      private_metadata: JSON.stringify({ channel_id: command.channel_id }),
      blocks: [ /* ... */ ],
    },
  });
});
```

## Parsing command text

```ts
app.command('/todo', async ({ command, ack, respond }) => {
  await ack();

  const [subcommand, ...args] = command.text.trim().split(/\s+/);

  switch (subcommand) {
    case 'add':
      await respond({ text: `Added: ${args.join(' ')}` });
      break;
    case 'list':
      await respond({ text: 'Your todos: ...' });
      break;
    case 'help':
    default:
      await respond({ text: 'Usage: /todo [add|list|help]' });
  }
});
```

## Escaped Parameters

When "Escape channels, users, and links" is enabled in app settings, Slack converts:

| Input | Converted to |
|-------|-------------|
| `@username` | `<@U012ABCDEF>` |
| `#channel` | `<#C012ABCDE\|channel-name>` |
| `https://example.com` | `<https://example.com>` |
| Private channel | `<C987654321\|>` (empty name) |

## Required Scope

Slash commands require the `commands` scope. No additional scope is needed to register commands — they are configured in your app's settings or manifest.

## Configuration Fields (App Manifest)

```json
{
  "command": "/weather",
  "description": "Get weather for a city",
  "usage_hint": "/weather [city]",
  "url": "https://your-app.com/slack/commands",
  "should_escape": true
}
```
