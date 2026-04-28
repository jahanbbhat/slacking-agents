# Slack block_actions Payload Reference

> Fetched from docs.slack.dev — 2026-04-27

Fired when a user interacts with any interactive element in a message or modal (button click, select change, checkbox toggle, date selection, etc.).

## Top-Level Payload Shape

```ts
interface BlockActionsPayload {
  type: 'block_actions';
  team: { id: string; domain: string };
  user: { id: string; username: string; team_id: string };
  api_app_id: string;
  token: string;                    // DEPRECATED — use signing secret
  trigger_id: string;               // use within 3 seconds to open a modal
  container: {
    type: 'message' | 'view';
    message_ts?: string;
    channel_id?: string;
    view_id?: string;
    is_ephemeral?: boolean;
  };
  channel?: { id: string; name: string };   // present for message interactions
  message?: { type: string; ts: string; text: string; blocks: any[] };
  view?: {                                  // present for modal/home interactions
    id: string;
    type: 'modal' | 'home';
    callback_id: string;
    hash: string;
    private_metadata: string;
    state: { values: Record<string, any> };
  };
  response_url?: string;            // only for message-based interactions
  hash: string;                     // use with views.update to prevent races
  actions: BlockAction[];
  state?: { values: Record<string, any> }; // all stateful element values
}
```

## The actions Array

Each action object varies by element type:

```ts
interface BlockAction {
  type: string;       // element type: "button", "static_select", "datepicker", etc.
  action_id: string;  // your action_id from the block definition
  block_id: string;   // your block_id from the block definition
  action_ts: string;  // unix timestamp of the interaction
}
```

## Reading Values by Element Type

```ts
app.action('my_action', async ({ action, ack }) => {
  await ack();

  // Button
  const buttonValue = (action as ButtonAction).value;

  // Static select / radio buttons
  const selected = (action as StaticSelectAction).selected_option?.value;

  // Multi-select
  const selections = (action as MultiStaticSelectAction).selected_options?.map(o => o.value);

  // Checkboxes
  const checked = (action as CheckboxesAction).selected_options?.map(o => o.value);

  // Date picker
  const date = (action as DatepickerAction).selected_date;  // "YYYY-MM-DD"

  // Time picker
  const time = (action as TimepickerAction).selected_time;  // "HH:mm"

  // Users select
  const userId = (action as UsersSelectAction).selected_user;

  // Conversations select
  const convId = (action as ConversationsSelectAction).selected_conversation;

  // Channels select
  const channelId = (action as ChannelsSelectAction).selected_channel;
});
```

## Handling by action_id

```ts
// Listen for a specific action
app.action('approve_button', async ({ action, ack, respond, body, client }) => {
  await ack();
  // handle...
});

// Listen with a regex
app.action(/^ticket_\d+$/, async ({ action, ack }) => {
  await ack();
});
```

## Common Response Patterns

### Open a modal (requires trigger_id — use within 3 seconds)

```ts
app.action('open_modal_btn', async ({ ack, body, client }) => {
  await ack();
  await client.views.open({
    trigger_id: body.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'my_modal',
      title: { type: 'plain_text', text: 'Details' },
      blocks: [],
    },
  });
});
```

### Update the message in place

```ts
app.action('update_btn', async ({ ack, body, client }) => {
  await ack();
  await client.chat.update({
    channel: body.channel!.id,
    ts: body.message!.ts,
    text: 'Updated!',
    blocks: [ /* new blocks */ ],
  });
});
```

### Respond via response_url (message context only)

```ts
app.action('respond_btn', async ({ ack, respond }) => {
  await ack();
  await respond({
    text: 'Got it!',
    replace_original: false,   // true to replace the original message
    response_type: 'ephemeral',
  });
});
```

### Update a modal view (use body.view.hash to prevent races)

```ts
app.action('refresh_view_btn', async ({ ack, body, client }) => {
  await ack();
  await client.views.update({
    view_id: body.view!.id,
    hash: body.view!.hash,
    view: {
      type: 'modal',
      callback_id: 'my_modal',
      title: { type: 'plain_text', text: 'Refreshed' },
      blocks: [ /* updated blocks */ ],
    },
  });
});
```

## Key Rules

- Always `ack()` within **3 seconds**
- `trigger_id` expires in **3 seconds** — call `views.open` before any async work
- `response_url` is only present for message-based interactions, not modal interactions
- Use `body.view.hash` when calling `views.update` to prevent overwriting concurrent edits
- Deselecting a static select returns `null` for `selected_option`
- `state.values` in the payload reflects all current input values in the view at time of action
