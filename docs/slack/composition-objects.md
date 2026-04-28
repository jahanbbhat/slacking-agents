# Slack Block Kit Composition Objects Reference

> Written from Slack API reference — 2026-04-27

Composition objects are reusable building blocks used as field values within blocks and elements. They are not standalone blocks.

---

## Text Object

Used wherever a text value is required. Two types: `plain_text` and `mrkdwn`.

```ts
// plain_text — no formatting, required for labels, titles, button text, hints
{ type: 'plain_text', text: 'Hello world', emoji: true }

// mrkdwn — supports formatting, mentions, links
{ type: 'mrkdwn', text: '*Bold* and _italic_ and <@U123|mention>', verbatim: false }
```

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"plain_text"` \| `"mrkdwn"` | |
| `text` | Yes | String | Max length varies by context (see block/element spec) |
| `emoji` | No | Boolean | `plain_text` only — allow emoji colon syntax; default `false` |
| `verbatim` | No | Boolean | `mrkdwn` only — skip auto-parsing of links/mentions; default `false` |

### Where each type is required

| Context | Required type |
|---------|--------------|
| `header` block text | `plain_text` |
| `input` block label | `plain_text` |
| `input` block hint | `plain_text` |
| Modal `title` | `plain_text` |
| Modal `submit` / `close` | `plain_text` |
| Button text | `plain_text` |
| Select placeholder | `plain_text` |
| `section` block text | Either |
| `context` block elements | Either |
| Option `text` | Either |
| Option `description` | Either |

---

## Option Object

Used in select menus, radio buttons, checkboxes, and overflow menus.

```ts
interface OptionObject {
  text: TextObject;          // display label; plain_text or mrkdwn
  value: string;             // returned in payload; max 150 chars
  description?: TextObject;  // secondary text below label; plain_text or mrkdwn; max 75 chars
  url?: string;              // overflow menu only — opens URL on click; max 3000 chars
}
```

```ts
// Basic option
{ text: { type: 'plain_text', text: 'Option A' }, value: 'a' }

// With description
{
  text: { type: 'plain_text', text: 'High Priority' },
  value: 'high',
  description: { type: 'mrkdwn', text: '_Notifies the on-call team_' },
}

// Overflow menu item with URL
{
  text: { type: 'plain_text', text: 'Open in Jira' },
  value: 'jira',
  url: 'https://jira.example.com/ticket/123',
}
```

### Constraints by element

| Element | Max options | text type |
|---------|-------------|-----------|
| `static_select` | 100 | `plain_text` or `mrkdwn` |
| `multi_static_select` | 100 | `plain_text` or `mrkdwn` |
| `radio_buttons` | 10 | `plain_text` |
| `checkboxes` | 10 | `plain_text` or `mrkdwn` |
| `overflow` | 5 | `plain_text` |

---

## Option Group Object

Groups options under a label in select menus.

```ts
interface OptionGroup {
  label: TextObject;    // plain_text only; max 75 chars
  options: OptionObject[];  // max 100 per group
}
```

```ts
{
  label: { type: 'plain_text', text: 'Engineering' },
  options: [
    { text: { type: 'plain_text', text: 'Frontend' }, value: 'fe' },
    { text: { type: 'plain_text', text: 'Backend' }, value: 'be' },
  ],
}
```

Use `option_groups` instead of `options` in a select element:

```ts
{
  type: 'static_select',
  action_id: 'team_select',
  placeholder: { type: 'plain_text', text: 'Select a team' },
  option_groups: [
    {
      label: { type: 'plain_text', text: 'Engineering' },
      options: [
        { text: { type: 'plain_text', text: 'Frontend' }, value: 'fe' },
        { text: { type: 'plain_text', text: 'Backend' }, value: 'be' },
      ],
    },
    {
      label: { type: 'plain_text', text: 'Design' },
      options: [
        { text: { type: 'plain_text', text: 'UX' }, value: 'ux' },
      ],
    },
  ],
}
```

---

## Confirm Object

An optional confirmation dialog shown before an action is submitted. Works with buttons, select menus, datepickers, and other interactive elements.

```ts
interface ConfirmObject {
  title: TextObject;    // plain_text; max 100 chars
  text: TextObject;     // plain_text or mrkdwn; max 300 chars
  confirm: TextObject;  // plain_text; label for the confirm button; max 30 chars
  deny: TextObject;     // plain_text; label for the cancel button; max 30 chars
  style?: 'primary' | 'danger';  // color of the confirm button
}
```

```ts
{
  type: 'button',
  text: { type: 'plain_text', text: 'Delete' },
  style: 'danger',
  value: 'delete',
  action_id: 'delete_btn',
  confirm: {
    title: { type: 'plain_text', text: 'Are you sure?' },
    text: { type: 'mrkdwn', text: 'This will *permanently delete* the record.' },
    confirm: { type: 'plain_text', text: 'Delete' },
    deny: { type: 'plain_text', text: 'Cancel' },
    style: 'danger',
  },
}
```

---

## Filter Object

Limits which conversation types appear in `conversations_select` and `multi_conversations_select` elements.

```ts
interface FilterObject {
  include?: Array<'im' | 'mpim' | 'private' | 'public'>;
  exclude_external_shared_channels?: boolean;  // default false
  exclude_bot_users?: boolean;                 // default false
}
```

```ts
// Only show public and private channels (exclude DMs)
{
  type: 'conversations_select',
  action_id: 'target_channel',
  filter: {
    include: ['public', 'private'],
    exclude_bot_users: true,
  },
}

// Only show DMs
{
  type: 'conversations_select',
  action_id: 'dm_target',
  filter: { include: ['im'] },
}
```

---

## Dispatch Action Config Object

Controls when a `plain_text_input` element fires a `block_actions` payload during typing. Used to build live-validation or typeahead patterns.

```ts
interface DispatchActionConfig {
  trigger_actions_on: Array<'on_enter_pressed' | 'on_character_entered'>;
}
```

```ts
{
  type: 'plain_text_input',
  action_id: 'search_input',
  dispatch_action_config: {
    trigger_actions_on: ['on_character_entered'],  // fires on every keystroke
  },
}

// In the parent input block, set dispatch_action: true
{
  type: 'input',
  dispatch_action: true,
  block_id: 'search_block',
  label: { type: 'plain_text', text: 'Search' },
  element: {
    type: 'plain_text_input',
    action_id: 'search_input',
    dispatch_action_config: {
      trigger_actions_on: ['on_character_entered'],
    },
  },
}
```

---

## Quick Reference

| Object | Used in |
|--------|---------|
| `text` (plain_text) | Block labels, modal titles, button text, placeholders, hints |
| `text` (mrkdwn) | Section text, context elements, option descriptions |
| `option` | static_select, multi_static_select, radio_buttons, checkboxes, overflow |
| `option_group` | static_select, multi_static_select, external_select (response) |
| `confirm` | Any interactive element to add a confirmation step |
| `filter` | conversations_select, multi_conversations_select |
| `dispatch_action_config` | plain_text_input (for live validation / typeahead) |
