# Bolt.js TypeScript Types Reference

> Written from @slack/bolt SDK — 2026-04-27

## Handler Argument Types

Every Bolt listener receives a single destructured object. The types vary by listener kind.

### app.command — SlashCommand

```ts
import { App, SlashCommand } from '@slack/bolt';

app.command('/create', async ({ command, ack, respond, say, client, logger, context }) => {
  // command: SlashCommand
});

interface SlashCommand {
  command: string;           // "/create"
  text: string;              // everything after the command
  user_id: string;
  user_name: string;         // deprecated
  channel_id: string;
  channel_name: string;
  team_id: string;
  team_domain: string;
  enterprise_id?: string;
  enterprise_name?: string;
  response_url: string;
  trigger_id: string;
  api_app_id: string;
}
```

### app.action — BlockAction

```ts
import { BlockAction, ButtonAction, StaticSelectAction, CheckboxesAction } from '@slack/bolt';

// Generic block action
app.action('my_action', async ({ action, body, ack, respond, client }) => {
  // action is typed based on what you narrow it to
  // body: BlockAction
});

// Narrowing to specific element types
app.action<BlockAction<ButtonAction>>('approve_btn', async ({ action }) => {
  const value: string = action.value;
});

app.action<BlockAction<StaticSelectAction>>('team_select', async ({ action }) => {
  const selected = action.selected_option?.value;
});

app.action<BlockAction<CheckboxesAction>>('options_check', async ({ action }) => {
  const checked = action.selected_options.map(o => o.value);
});
```

**Common action subtypes:**

| Import | Element | Key field |
|--------|---------|-----------|
| `ButtonAction` | button | `value: string` |
| `StaticSelectAction` | static_select | `selected_option: Option` |
| `MultiStaticSelectAction` | multi_static_select | `selected_options: Option[]` |
| `UsersSelectAction` | users_select | `selected_user: string` |
| `MultiUsersSelectAction` | multi_users_select | `selected_users: string[]` |
| `ConversationsSelectAction` | conversations_select | `selected_conversation: string` |
| `ChannelsSelectAction` | channels_select | `selected_channel: string` |
| `DatepickerAction` | datepicker | `selected_date: string` (YYYY-MM-DD) |
| `TimepickerAction` | timepicker | `selected_time: string` (HH:mm) |
| `RadioButtonsAction` | radio_buttons | `selected_option: Option` |
| `CheckboxesAction` | checkboxes | `selected_options: Option[]` |
| `OverflowAction` | overflow | `selected_option: Option` |
| `PlainTextInputAction` | plain_text_input | `value: string` |

### app.view — ViewSubmitAction / ViewClosedAction

```ts
import { ViewSubmitAction, ViewClosedAction } from '@slack/bolt';

// Submission
app.view('my_modal', async ({ view, body, ack, client, context }) => {
  // body: ViewSubmitAction
  // view: body.view (the submitted view object)
  const vals = view.state.values;
  const text = vals['block_id']['action_id'].value;
});

// Closed (notify_on_close: true)
app.view({ callback_id: 'my_modal', type: 'view_closed' }, async ({ view, body, ack }) => {
  // body: ViewClosedAction
  await ack();
});
```

### app.event — AppMentionEvent / MessageEvent

```ts
import { AppMentionEvent, GenericMessageEvent } from '@slack/bolt';

app.event('app_mention', async ({ event, say, client }) => {
  // event: AppMentionEvent
  const { user, text, ts, channel } = event;
});

app.event('message', async ({ event, client }) => {
  // event: GenericMessageEvent (for regular messages)
  if ((event as any).bot_id) return; // skip bot messages

  const msgEvent = event as GenericMessageEvent;
  const { user, text, ts, channel, thread_ts } = msgEvent;
});
```

### app.shortcut

```ts
import { GlobalShortcut, MessageShortcut } from '@slack/bolt';

app.shortcut('global_shortcut_id', async ({ shortcut, ack, client }) => {
  // shortcut: GlobalShortcut
  const { trigger_id, user, team } = shortcut as GlobalShortcut;
});

app.shortcut('message_shortcut_id', async ({ shortcut, ack, client }) => {
  // shortcut: MessageShortcut
  const { trigger_id, message, channel, response_url } = shortcut as MessageShortcut;
});
```

## Utility Function Types

### SayFn

Posts a message to the channel associated with the incoming event.

```ts
import { SayFn, SayArguments } from '@slack/bolt';

app.event('app_mention', async ({ event, say }) => {
  // Simple string
  await say('Hello!');

  // SayArguments — same as chat.postMessage args, channel pre-filled
  await say({
    text: 'Hello with blocks',
    blocks: [ /* ... */ ],
    thread_ts: event.ts,  // reply in thread
  });
});
```

### RespondFn

Sends a message via `response_url`. Available in command and action handlers.

```ts
import { RespondFn, RespondArguments } from '@slack/bolt';

app.command('/ping', async ({ ack, respond }) => {
  await ack();
  await respond('Pong!');

  // RespondArguments
  await respond({
    text: 'Pong!',
    response_type: 'in_channel',
    replace_original: false,
    delete_original: false,
  });
});
```

### AckFn

Acknowledges an incoming interaction. Must be called within 3 seconds.

```ts
// Simple ack (closes interaction)
await ack();

// Ack with immediate message (commands only)
await ack({ response_type: 'ephemeral', text: 'Got it!' });

// Ack with response_action (view submissions)
await ack({ response_action: 'errors', errors: { block_id: 'Error message' } });
await ack({ response_action: 'clear' });
await ack({ response_action: 'update', view: { /* ... */ } });
await ack({ response_action: 'push', view: { /* ... */ } });
```

## Context Object

Available in all handlers. Contains token and workspace info.

```ts
interface Context {
  botToken: string;          // xoxb-... for current workspace
  botId: string;
  botUserId: string;
  teamId: string;
  enterpriseId?: string;
  isEnterpriseInstall: boolean;
  // Any values added by middleware via context.key = value
}
```

```ts
app.use(async ({ context, next }) => {
  context.requestId = crypto.randomUUID(); // add custom values
  await next();
});

app.command('/ping', async ({ context }) => {
  console.log(context.requestId); // available downstream
});
```

## Typing view.state.values

`view.state.values` is typed as `Record<string, Record<string, any>>` by default. Cast to be explicit:

```ts
interface MyModalState {
  title_block: { title_input: { type: 'plain_text_input'; value: string | null } };
  priority_block: { priority_radio: { type: 'radio_buttons'; selected_option: { value: string } | null } };
  assignee_block: { assignee_select: { type: 'users_select'; selected_user: string | null } };
}

app.view('my_modal', async ({ view, ack }) => {
  await ack();
  const vals = view.state.values as unknown as MyModalState;
  const title = vals.title_block.title_input.value ?? '';
  const priority = vals.priority_block.priority_radio.selected_option?.value;
  const assignee = vals.assignee_block.assignee_select.selected_user;
});
```
