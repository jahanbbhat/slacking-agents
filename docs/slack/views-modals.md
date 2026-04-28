# Slack Views & Modals API Reference

> Fetched from docs.slack.dev — 2026-04-27

## View Object Fields

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"modal"` \| `"home"` | |
| `title` | Yes (modal) | `plain_text` object | Max 24 chars |
| `blocks` | Yes | Block[] | Max 100 blocks |
| `callback_id` | No | String | Identifies this view in `view_submission`/`view_closed` payloads; max 255 chars |
| `submit` | Conditional | `plain_text` object | Required if any `input` blocks present; max 24 chars |
| `close` | No | `plain_text` object | Close button label; max 24 chars; modals only |
| `private_metadata` | No | String | Hidden string passed through all payloads; max 3000 chars |
| `clear_on_close` | No | Boolean | Close button dismisses entire modal stack |
| `notify_on_close` | No | Boolean | Fires `view_closed` event when user dismisses |
| `external_id` | No | String | Unique identifier per team; use to update without a `view_id` |
| `submit_disabled` | No | Boolean | Disables submit until all required inputs are filled |

## API Methods

### views.open
Opens a new modal. Requires a `trigger_id` from an interaction (expires in **3 seconds**).

```ts
app.command('/create', async ({ command, ack, client }) => {
  await ack();
  await client.views.open({
    trigger_id: command.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'create_modal',
      title: { type: 'plain_text', text: 'Create Item' },
      submit: { type: 'plain_text', text: 'Create' },
      close: { type: 'plain_text', text: 'Cancel' },
      private_metadata: JSON.stringify({ channel: command.channel_id }),
      notify_on_close: true,
      blocks: [ /* ... */ ],
    },
  });
});
```

### views.push
Pushes a new view onto the modal stack. Requires a `trigger_id` from a `block_actions` payload. **Maximum stack depth: 3 views.**

```ts
app.action('next_step_btn', async ({ ack, body, client }) => {
  await ack();
  await client.views.push({
    trigger_id: body.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'step_two_modal',
      title: { type: 'plain_text', text: 'Step 2' },
      submit: { type: 'plain_text', text: 'Submit' },
      blocks: [ /* ... */ ],
    },
  });
});
```

### views.update
Updates an existing view in place. Use `hash` from the payload to prevent race conditions.

```ts
app.action('refresh_btn', async ({ ack, body, client }) => {
  await ack();
  await client.views.update({
    view_id: body.view.id,
    hash: body.view.hash,
    view: {
      type: 'modal',
      callback_id: 'create_modal',
      title: { type: 'plain_text', text: 'Create Item' },
      submit: { type: 'plain_text', text: 'Create' },
      blocks: [ /* updated blocks */ ],
    },
  });
});
```

### views.publish
Publishes or updates the App Home tab for a specific user. No `trigger_id` required.

```ts
app.event('app_home_opened', async ({ event, client }) => {
  await client.views.publish({
    user_id: event.user,
    view: {
      type: 'home',
      blocks: [ /* home tab blocks */ ],
    },
  });
});
```

## Handling view_submission

```ts
app.view('create_modal', async ({ view, ack, body, client }) => {
  // Extract values
  const vals = view.state.values;
  const title = vals['title_block']['title_input'].value;
  const meta = JSON.parse(view.private_metadata ?? '{}');

  // Option 1: close the modal
  await ack();

  // Option 2: update the modal in place
  await ack({
    response_action: 'update',
    view: { type: 'modal', title: { type: 'plain_text', text: 'Done!' }, blocks: [] },
  });

  // Option 3: push another view
  await ack({
    response_action: 'push',
    view: {
      type: 'modal',
      callback_id: 'confirm_modal',
      title: { type: 'plain_text', text: 'Confirm' },
      submit: { type: 'plain_text', text: 'Yes' },
      blocks: [],
    },
  });

  // Option 4: show validation errors (does not close modal)
  await ack({
    response_action: 'errors',
    errors: {
      title_block: 'Title must be at least 3 characters',
    },
  });

  // Option 5: close all views
  await ack({ response_action: 'clear' });
});
```

## view_submission Payload Shape

```ts
interface ViewSubmissionPayload {
  type: 'view_submission';
  user: { id: string; username: string; name: string; team_id: string };
  team: { id: string; domain: string };
  trigger_id: string;
  view: {
    id: string;
    team_id: string;
    type: 'modal';
    callback_id: string;
    private_metadata: string;
    hash: string;
    state: {
      values: {
        [blockId: string]: {
          [actionId: string]: {
            type: string;
            value?: string;                    // plain_text_input
            selected_option?: { value: string }; // static_select, radio_buttons
            selected_options?: { value: string }[]; // checkboxes, multi_select
            selected_date?: string;             // datepicker (YYYY-MM-DD)
            selected_time?: string;             // timepicker (HH:MM)
            selected_user?: string;             // users_select
            selected_conversation?: string;     // conversations_select
            selected_channel?: string;          // channels_select
          };
        };
      };
    };
    root_view_id: string;
    previous_view_id: string | null;
    app_id: string;
    external_id: string;
  };
  response_urls: Array<{ block_id: string; action_id: string; channel_id: string; response_url: string }>;
}
```

## view_closed Payload Shape

Only fired when `notify_on_close: true` is set on the view.

```ts
interface ViewClosedPayload {
  type: 'view_closed';
  user: { id: string; username: string; name: string; team_id: string };
  team: { id: string; domain: string };
  view: { id: string; callback_id: string; private_metadata: string; /* ... */ };
  is_cleared: boolean;   // true if clear_on_close triggered the dismissal
}
```

```ts
app.view({ callback_id: 'create_modal', type: 'view_closed' }, async ({ view, ack }) => {
  await ack();
  // Clean up any draft state
});
```

## Key Constraints

| Constraint | Limit |
|-----------|-------|
| `trigger_id` expires | 3 seconds after interaction |
| Modal stack depth | 3 views max |
| `title` / `submit` / `close` text | 24 chars max |
| `callback_id` | 255 chars max |
| `private_metadata` | 3000 chars max |
| Blocks per view | 100 max |
| `response_action` response window | 3 seconds |

## private_metadata Patterns

```ts
// Store context across a multi-step modal flow
const meta = JSON.stringify({ channelId: 'C123', userId: 'U456', draftId: 42 });

// Read it back in view_submission
const { channelId, userId, draftId } = JSON.parse(view.private_metadata ?? '{}');
```
