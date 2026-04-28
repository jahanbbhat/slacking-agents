# Slack mrkdwn Formatting Reference

> Fetched from docs.slack.dev — 2026-04-27

mrkdwn is Slack's text formatting syntax. It is similar to Markdown but not identical — do not assume standard Markdown rules apply.

## Basic Styling

| Effect | Syntax | Example |
|--------|--------|---------|
| Bold | `*text*` | `*Important*` → **Important** |
| Italic | `_text_` | `_note_` → _note_ |
| Strikethrough | `~text~` | `~cancelled~` → ~~cancelled~~ |
| Inline code | `` `text` `` | `` `npm install` `` |
| Code block | ` ```text``` ` | Multi-line preformatted block |

```ts
// In a Block Kit text object
{ type: 'mrkdwn', text: '*Bold*, _italic_, ~struck~, `code`' }
```

## Structure

```
Block quote:    > This is quoted text

Lists (no native list syntax — simulate with dashes and \n):
- Item one\n- Item two\n- Item three

Ordered lists (simulated):
1. First\n2. Second\n3. Third

Line break:     Use \n in the string value
```

> **Note:** mrkdwn does not support nested lists or true ordered list syntax. Use `\n` + dash/number prefixes.

## Links

```
URL (auto-display):         <https://example.com>
URL with label:             <https://example.com|Click here>
Email:                      <mailto:user@example.com|Email us>
```

```ts
{ type: 'mrkdwn', text: 'See <https://docs.slack.dev|Slack Docs> for details.' }
```

## Mentions

| Target | Syntax | Notes |
|--------|--------|-------|
| User | `<@U012AB3CD>` | Sends a notification |
| Channel | `<#C123ABC456>` or `<#C123ABC456\|channel-name>` | Links the channel |
| User group | `<!subteam^GROUPID>` | Notifies the group |
| @here | `<!here>` | Notifies active members |
| @channel | `<!channel>` | Notifies all members |
| @everyone | `<!everyone>` | Workspace-wide (admin only) |

```ts
{ type: 'mrkdwn', text: `Hey <@${userId}>, your ticket was assigned to <#${channelId}>.` }
```

## Date Formatting

Renders timestamps in the user's local timezone automatically.

```
Syntax: <!date^UNIX_TIMESTAMP^TOKEN_STRING^OPTIONAL_URL|FALLBACK_TEXT>
```

| Token | Example output |
|-------|---------------|
| `{date_num}` | 2026-04-27 |
| `{date}` | April 27th, 2026 |
| `{date_short}` | Apr 27, 2026 |
| `{date_long}` | Monday, April 27th, 2026 |
| `{date_pretty}` | today / yesterday / tomorrow |
| `{time}` | 9:30 AM |
| `{time_secs}` | 9:30:45 AM |
| `{ago}` | 3 minutes ago |

```ts
const ts = Math.floor(Date.now() / 1000);
const formatted = `<!date^${ts}^{date_pretty} at {time}|${new Date().toISOString()}>`;
// Renders as "today at 9:30 AM" in the user's timezone
```

Multiple tokens can be combined: `{date_short} at {time}` → `Apr 27, 2026 at 9:30 AM`

## Character Escaping

These characters have special meaning in mrkdwn and must be escaped when used literally:

| Character | Escape as |
|-----------|-----------|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |

```ts
function escapeMrkdwn(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;');
}
```

## Emoji

Use colon syntax: `:tada:`, `:white_check_mark:`, `:warning:`, `:x:`

Unicode emoji are auto-converted to colon format by Slack.

## Disabling Formatting

```ts
// plain_text block — no formatting applied
{ type: 'plain_text', text: 'Raw text, *not bold*' }

// Disable mrkdwn on an entire message
await client.chat.postMessage({ channel, text: 'No *formatting*', mrkdwn: false });

// Disable auto-parsing in a block text object
{ type: 'mrkdwn', text: 'URL not auto-linked', verbatim: true }
```

## Common Patterns

```ts
// Status message with emoji, mention, and timestamp
const ts = Math.floor(Date.now() / 1000);
const text = [
  `:white_check_mark: *Deployment complete*`,
  `Triggered by <@${userId}> at <!date^${ts}^{time}|${new Date().toTimeString()}>`,
  `Environment: \`production\``,
].join('\n');

// Always include a text fallback when using blocks
await client.chat.postMessage({
  channel,
  text: 'Deployment complete',   // shown in notifications and accessibility
  blocks: [ /* rich layout */ ],
});
```

## mrkdwn vs plain_text

| | `mrkdwn` | `plain_text` |
|--|---------|-------------|
| Bold/italic | Yes | No |
| Links | Yes | No |
| Mentions | Yes | No |
| Emoji colon syntax | Yes | Yes (with `emoji: true`) |
| Use in `header` block | No | Required |
| Use in `input` labels/hints | No | Required |
| Use in button/select text | No | Required |
| Use in modal title/submit/close | No | Required |
