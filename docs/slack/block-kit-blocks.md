# Block Kit Blocks Reference

> Fetched from docs.slack.dev — 2026-04-27

## Global Constraints

- Max 50 blocks per message
- Max 100 blocks per modal or Home tab
- `block_id`: max 255 characters; must be unique per message/view iteration
- Always set a `text` fallback on messages that use `blocks` (for notifications and accessibility)

## Surface Compatibility

| Block | Messages | Modals | Home Tab |
|-------|----------|--------|----------|
| `actions` | Yes | Yes | Yes |
| `context` | Yes | Yes | Yes |
| `divider` | Yes | Yes | Yes |
| `file` | Yes | No | No |
| `header` | Yes | Yes | Yes |
| `image` | Yes | Yes | Yes |
| `input` | No | Yes | Yes |
| `rich_text` | Yes | No | Yes |
| `section` | Yes | Yes | Yes |
| `video` | Yes | No | Yes |

## Block Specifications

### `section`
A flexible text block with optional fields and accessory element.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"section"` | |
| `text` | Conditional | Text object | Required if no `fields`; max 3000 chars |
| `block_id` | No | String | Max 255 chars |
| `fields` | No | Text object[] | Max 10 items; 2000 chars each; renders 2-column |
| `accessory` | No | Block element | One interactive element beside the text |
| `expand` | No | Boolean | Expands long text without user interaction |

```json
{
  "type": "section",
  "text": { "type": "mrkdwn", "text": "*Bold* and _italic_ text." },
  "accessory": {
    "type": "button",
    "text": { "type": "plain_text", "text": "Click me" },
    "action_id": "button_click",
    "value": "click_value"
  }
}
```

### `actions`
A block containing interactive elements.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"actions"` | |
| `elements` | Yes | Element[] | Max 25 elements |
| `block_id` | No | String | Max 255 chars |

Supported elements: buttons, static/external/users/conversations/channels select menus, overflow menus, date pickers, time pickers, radio buttons, checkboxes.

```json
{
  "type": "actions",
  "block_id": "my_actions",
  "elements": [
    {
      "type": "button",
      "text": { "type": "plain_text", "text": "Approve" },
      "style": "primary",
      "value": "approve",
      "action_id": "approve_button"
    },
    {
      "type": "button",
      "text": { "type": "plain_text", "text": "Deny" },
      "style": "danger",
      "value": "deny",
      "action_id": "deny_button"
    }
  ]
}
```

### `input`
A labeled form element. Only valid in modals and Home tabs.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"input"` | |
| `label` | Yes | `plain_text` object | Max 2000 chars |
| `element` | Yes | Block element | Input-compatible elements only |
| `block_id` | No | String | Max 255 chars |
| `hint` | No | `plain_text` object | Helper text below input; max 2000 chars |
| `optional` | No | Boolean | If true, field may be empty on submit; default false |
| `dispatch_action` | No | Boolean | If true, fires `block_actions` on change; default false; incompatible with `file_input` |

```json
{
  "type": "input",
  "block_id": "title_input",
  "label": { "type": "plain_text", "text": "Title" },
  "element": { "type": "plain_text_input", "action_id": "title_value" },
  "hint": { "type": "plain_text", "text": "Keep it short." },
  "optional": false
}
```

### `header`
A large-text block for section headings.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"header"` | |
| `text` | Yes | `plain_text` object | Max 150 chars; no mrkdwn |
| `block_id` | No | String | Max 255 chars |

### `context`
Displays secondary contextual info — images and/or text, rendered small.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"context"` | |
| `elements` | Yes | Element[] | Max 10 elements; image or text objects only |
| `block_id` | No | String | Max 255 chars |

### `divider`
A horizontal rule. No fields other than `type` and optional `block_id`.

```json
{ "type": "divider" }
```

### `image`
A standalone image block.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"image"` | |
| `image_url` | Yes | String | Public URL of the image |
| `alt_text` | Yes | String | Accessibility description; max 2000 chars |
| `title` | No | `plain_text` object | Max 2000 chars |
| `block_id` | No | String | Max 255 chars |

### `rich_text`
Structured rich text. Not available in modals.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"rich_text"` | |
| `elements` | Yes | Rich text element[] | `rich_text_section`, `rich_text_list`, `rich_text_preformatted`, `rich_text_quote` |
| `block_id` | No | String | Max 255 chars |

### `file`
Displays an existing Slack-hosted file. Messages only.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"file"` | |
| `external_id` | Yes | String | The file's external ID |
| `source` | Yes | `"remote"` | Always `"remote"` |
| `block_id` | No | String | Max 255 chars |

### `video`
Embeds a video. Not available in modals.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | Yes | `"video"` | |
| `video_url` | Yes | String | URL of the video |
| `thumbnail_url` | Yes | String | Preview image URL |
| `alt_text` | Yes | String | Accessibility text |
| `title` | Yes | `plain_text` object | Max 200 chars |
| `block_id` | No | String | Max 255 chars |
| `title_url` | No | String | Makes title a hyperlink |
| `description` | No | `plain_text` object | Caption text |
| `author_name` | No | String | Max 50 chars |
| `provider_name` | No | String | e.g. "YouTube" |
| `provider_icon_url` | No | String | Provider logo URL |
