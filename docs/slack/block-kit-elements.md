# Block Kit Interactive Elements Reference

> Fetched from docs.slack.dev — 2026-04-27

Interactive elements go inside `actions` blocks or `input` blocks. A few (button, overflow, image) can also appear as the `accessory` of a `section` block.

**Global rules:**
- `action_id`: max 255 characters; must be unique per view
- `focus_on_load`: only one element per view may be `true`
- `placeholder`: always a `plain_text` object; max 150 characters
- All elements fire a `block_actions` payload when interacted with (unless inside an `input` block with `dispatch_action: false`)

---

## Button

Works in: `actions`, `section` (accessory)

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"button"` | |
| `text` | Yes | `plain_text` object | Max 75 chars |
| `action_id` | No | String | Max 255 chars |
| `value` | No | String | Sent in payload; max 2000 chars |
| `url` | No | String | Opens in browser on click; still fires payload; max 3000 chars |
| `style` | No | `"primary"` \| `"danger"` | `primary` = green; `danger` = red |
| `confirm` | No | Confirm object | Confirmation dialog |
| `accessibility_label` | No | String | Screen reader label; max 75 chars |

```ts
{
  type: 'button',
  text: { type: 'plain_text', text: 'Approve' },
  style: 'primary',
  value: 'approve',
  action_id: 'approve_btn',
}
```

---

## Plain Text Input

Works in: `input` only

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"plain_text_input"` | |
| `action_id` | No | String | Max 255 chars |
| `initial_value` | No | String | Pre-filled text |
| `multiline` | No | Boolean | `true` = textarea; default `false` |
| `min_length` | No | Integer | 0–3000 |
| `max_length` | No | Integer | 1–3000 |
| `placeholder` | No | `plain_text` object | Max 150 chars |
| `focus_on_load` | No | Boolean | Auto-focus; one per view |
| `dispatch_action_config` | No | Object | Fires `block_actions` on input events |

```ts
{
  type: 'plain_text_input',
  action_id: 'title_input',
  placeholder: { type: 'plain_text', text: 'Enter a title...' },
  multiline: false,
  max_length: 200,
}
```

**Extracting value on submission:**
```ts
app.view('my_modal', async ({ view, ack }) => {
  await ack();
  const title = view.state.values['title_block']['title_input'].value;
});
```

---

## Radio Buttons

Works in: `actions`, `input`, `section` (accessory)

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"radio_buttons"` | |
| `options` | Yes | Option[] | Max 10 options |
| `action_id` | No | String | Max 255 chars |
| `initial_option` | No | Option object | Must match one in `options` |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

```ts
{
  type: 'radio_buttons',
  action_id: 'priority_select',
  options: [
    { value: 'high', text: { type: 'plain_text', text: 'High' } },
    { value: 'medium', text: { type: 'plain_text', text: 'Medium' } },
    { value: 'low', text: { type: 'plain_text', text: 'Low' } },
  ],
  initial_option: { value: 'medium', text: { type: 'plain_text', text: 'Medium' } },
}
```

---

## Checkboxes

Works in: `actions`, `input`, `section` (accessory)

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"checkboxes"` | |
| `options` | Yes | Option[] | Max 10 options |
| `action_id` | No | String | Max 255 chars |
| `initial_options` | No | Option[] | Must match items in `options` |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

Options can include an optional `description` (`mrkdwn` or `plain_text` object).

```ts
{
  type: 'checkboxes',
  action_id: 'notify_options',
  options: [
    { value: 'email', text: { type: 'plain_text', text: 'Email' } },
    {
      value: 'slack',
      text: { type: 'plain_text', text: 'Slack DM' },
      description: { type: 'mrkdwn', text: '_Sends a direct message_' },
    },
  ],
  initial_options: [{ value: 'slack', text: { type: 'plain_text', text: 'Slack DM' } }],
}
```

---

## Date Picker

Works in: `actions`, `input`, `section` (accessory)

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"datepicker"` | |
| `action_id` | No | String | Max 255 chars |
| `initial_date` | No | String | Format: `YYYY-MM-DD` |
| `placeholder` | No | `plain_text` object | Max 150 chars |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

```ts
{
  type: 'datepicker',
  action_id: 'due_date',
  initial_date: '2026-05-01',
  placeholder: { type: 'plain_text', text: 'Select a date' },
}
```

---

## Static Select

Works in: `actions`, `input`, `section` (accessory)

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"static_select"` | |
| `options` or `option_groups` | Yes | Option[] | Max 100 options |
| `action_id` | No | String | Max 255 chars |
| `initial_option` | No | Option object | Must match one in `options` |
| `placeholder` | No | `plain_text` object | Max 150 chars |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

```ts
{
  type: 'static_select',
  action_id: 'team_select',
  placeholder: { type: 'plain_text', text: 'Select a team' },
  options: [
    { text: { type: 'plain_text', text: 'Engineering' }, value: 'eng' },
    { text: { type: 'plain_text', text: 'Design' }, value: 'design' },
  ],
}
```

---

## Users Select

Works in: `actions`, `input`, `section` (accessory)

Populated automatically with workspace members.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"users_select"` | |
| `action_id` | No | String | |
| `initial_user` | No | String | User ID |
| `placeholder` | No | `plain_text` object | |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

```ts
{ type: 'users_select', action_id: 'assignee', placeholder: { type: 'plain_text', text: 'Select a user' } }
```

---

## Conversations Select

Works in: `actions`, `input`, `section` (accessory)

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"conversations_select"` | |
| `action_id` | No | String | |
| `initial_conversation` | No | String | Conversation ID |
| `default_to_current_conversation` | No | Boolean | Pre-fills with current channel |
| `response_url_enabled` | No | Boolean | In `input` blocks only; provides `response_url` for async replies |
| `filter` | No | Filter object | Limit by type: `im`, `mpim`, `private`, `public` |
| `placeholder` | No | `plain_text` object | |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

---

## Channels Select

Works in: `actions`, `input`, `section` (accessory)

Like `conversations_select` but scoped to public channels only.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"channels_select"` | |
| `action_id` | No | String | |
| `initial_channel` | No | String | Channel ID |
| `response_url_enabled` | No | Boolean | `input` blocks only |
| `placeholder` | No | `plain_text` object | |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

---

## External Select

Works in: `actions`, `input`, `section` (accessory)

Loads options dynamically from your own endpoint. Requires "Options Load URL" configured in App Management > Interactivity & Shortcuts.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"external_select"` | |
| `action_id` | No | String | |
| `min_query_length` | No | Integer | Min characters before request fires; default 3 |
| `initial_option` | No | Option object | |
| `placeholder` | No | `plain_text` object | |
| `confirm` | No | Confirm object | |
| `focus_on_load` | No | Boolean | |

Your endpoint must respond within 1 second with:
```ts
{ options: [{ text: { type: 'plain_text', text: 'Option' }, value: 'val' }] }
```

---

## Extracting Values from view.state.values

```ts
app.view('my_modal', async ({ view, ack }) => {
  await ack();
  const vals = view.state.values;

  const text      = vals['text_block']['text_input'].value;
  const date      = vals['date_block']['date_picker'].selected_date;
  const user      = vals['user_block']['user_select'].selected_user;
  const channel   = vals['chan_block']['chan_select'].selected_channel;
  const option    = vals['radio_block']['radio_btns'].selected_option?.value;
  const options   = vals['check_block']['checkboxes'].selected_options?.map(o => o.value);
  const selection = vals['select_block']['static_sel'].selected_option?.value;
  const multiSel  = vals['multi_block']['multi_sel'].selected_options?.map(o => o.value);
  const multiUsers = vals['muser_block']['multi_users'].selected_users;
});
```

---

## Multi-Select Elements

Multi-selects allow users to pick multiple items. All variants share the same optional fields: `action_id`, `max_selected_items`, `confirm`, `focus_on_load`, `placeholder`.

Works in: `actions`, `input`, `section` (accessory)

**`max_selected_items`**: Minimum value 1; no maximum documented.

### multi_static_select

```ts
{
  type: 'multi_static_select',
  action_id: 'notify_channels',
  placeholder: { type: 'plain_text', text: 'Select channels to notify' },
  max_selected_items: 3,
  options: [
    { text: { type: 'plain_text', text: 'Engineering' }, value: 'eng' },
    { text: { type: 'plain_text', text: 'Design' }, value: 'design' },
    { text: { type: 'plain_text', text: 'Product' }, value: 'product' },
  ],
  initial_options: [
    { text: { type: 'plain_text', text: 'Engineering' }, value: 'eng' },
  ],
}
```

### multi_users_select

```ts
{
  type: 'multi_users_select',
  action_id: 'assignees',
  placeholder: { type: 'plain_text', text: 'Select assignees' },
  initial_users: ['U0USER001', 'U0USER002'],
}
```

### multi_conversations_select

```ts
{
  type: 'multi_conversations_select',
  action_id: 'target_convs',
  placeholder: { type: 'plain_text', text: 'Select conversations' },
  default_to_current_conversation: true,
  filter: { include: ['public', 'private'] },  // exclude 'im', 'mpim' etc.
  max_selected_items: 5,
}
```

### multi_channels_select

Public channels only.

```ts
{
  type: 'multi_channels_select',
  action_id: 'broadcast_channels',
  placeholder: { type: 'plain_text', text: 'Select channels' },
  initial_channels: ['C0CHANNEL01'],
}
```

### multi_external_select

Loads options dynamically from your endpoint. Requires Options Load URL in App Management.

```ts
{
  type: 'multi_external_select',
  action_id: 'dynamic_options',
  placeholder: { type: 'plain_text', text: 'Search...' },
  min_query_length: 2,
  max_selected_items: 10,
}
```

### Extracting multi-select values from view.state.values

```ts
app.view('my_modal', async ({ view, ack }) => {
  await ack();
  const vals = view.state.values;

  // multi_static_select → selected_options[]
  const channels = vals['chan_block']['notify_channels'].selected_options?.map(o => o.value) ?? [];

  // multi_users_select → selected_users[]
  const assignees = vals['user_block']['assignees'].selected_users ?? [];

  // multi_conversations_select → selected_conversations[]
  const convs = vals['conv_block']['target_convs'].selected_conversations ?? [];

  // multi_channels_select → selected_channels[]
  const bcastChannels = vals['bcast_block']['broadcast_channels'].selected_channels ?? [];
});
```
