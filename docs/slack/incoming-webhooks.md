# Slack Incoming Webhooks Reference

> Fetched from docs.slack.dev — 2026-04-27

Incoming webhooks are the simplest way to post messages to Slack — no event listeners, no OAuth token management per-message. A webhook URL is tied to a single channel.

## Setup

1. Enable **Incoming Webhooks** in your app settings
2. Click **Add New Webhook to Workspace** and select a channel
3. Copy the generated webhook URL — treat it as a secret

### Via OAuth (programmatic)

Add `incoming-webhook` to your `scope` param. After token exchange, the response includes:

```ts
interface OAuthResponse {
  ok: boolean;
  access_token: string;
  incoming_webhook: {
    channel: string;            // "#channel-name"
    channel_id: string;         // "C05002EAE"
    configuration_url: string;
    url: string;                // store and use this
  };
}
```

## Webhook URL Format

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

Never commit this URL to version control — Slack actively scans for and revokes leaked webhook URLs.

## Payload

```ts
interface WebhookPayload {
  text?: string;        // required if no blocks; shown in notifications
  blocks?: Block[];     // Block Kit blocks for rich formatting
  attachments?: any[];  // legacy; max 100; prefer blocks
  thread_ts?: string;   // reply in a thread
  mrkdwn?: boolean;     // default true
}
```

Always include `text` as a fallback even when using `blocks` — it's used in push notifications and accessibility.

## Sending Messages

### Simple text

```ts
const WEBHOOK_URL = process.env.SLACK_WEBHOOK_URL!;

await fetch(WEBHOOK_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ text: 'Hello from the app!' }),
});
```

### Rich message with Block Kit

```ts
await fetch(WEBHOOK_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    text: 'Deployment complete',   // fallback
    blocks: [
      {
        type: 'header',
        text: { type: 'plain_text', text: ':white_check_mark: Deployment complete' },
      },
      {
        type: 'section',
        fields: [
          { type: 'mrkdwn', text: '*Environment:*\nProduction' },
          { type: 'mrkdwn', text: '*Duration:*\n2m 14s' },
        ],
      },
    ],
  }),
});
```

### Using IncomingWebhook from @slack/webhook

```ts
import { IncomingWebhook } from '@slack/webhook';

const webhook = new IncomingWebhook(process.env.SLACK_WEBHOOK_URL!);

await webhook.send({
  text: 'Deployment complete',
  blocks: [ /* ... */ ],
});
```

### Threaded reply

```ts
// You need the original message ts — webhooks don't return it
// Retrieve it via conversations.history if needed
await fetch(WEBHOOK_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    text: 'Follow-up in thread',
    thread_ts: '1503435956.000247',
  }),
});
```

## Limitations

| Constraint | Detail |
|-----------|--------|
| Channel | Fixed at install time; cannot post to other channels |
| Custom username/icon | Not supported |
| Message deletion | Not possible via webhook |
| `message_ts` returned | No — cannot retrieve ts from webhook responses |
| Attachments | Max 100 per message (prefer blocks) |
| Rate limit | Same as `chat.postMessage` — 1 msg/sec per channel |

## Error Responses

| Error | Cause |
|-------|-------|
| `invalid_token` | Webhook URL expired or revoked |
| `channel_is_archived` | Cannot post to archived channel |
| `invalid_payload` | Malformed JSON |
| `no_text` | Missing `text` with no `blocks` |
| `too_many_attachments` | Exceeded 100 attachment limit |
| `action_prohibited` | Admin restrictions |

## When to Use Webhooks vs chat.postMessage

| Use webhooks when... | Use chat.postMessage when... |
|----------------------|------------------------------|
| Posting to one fixed channel | Posting to dynamic channels |
| No interactivity needed | Buttons/modals required |
| Simple CI/CD alerts | Message updates or deletions needed |
| No bot token available | Need the message ts back |
