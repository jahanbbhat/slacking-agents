# Slack Events API Reference

> Fetched from docs.slack.dev — 2026-04-27

## How Events Work

Events are delivered via HTTP POST to your Request URL or over a Socket Mode WebSocket connection. Every delivery must be acknowledged with HTTP 200 within 3 seconds.

URL verification: when you first configure your Request URL, Slack sends a `url_verification` challenge. Respond immediately with `{"challenge": "<challenge_value>"}`.

## Event Envelope Shape

```json
{
  "token": "DEPRECATED_VERIFICATION_TOKEN",
  "team_id": "T0EXAMPLE",
  "api_app_id": "A0EXAMPLE",
  "event": { ... },
  "type": "event_callback",
  "event_id": "Ev0EXAMPLE",
  "event_time": 1234567890,
  "authorizations": [{ "enterprise_id": null, "team_id": "T0EXAMPLE", "user_id": "U0EXAMPLE", "is_bot": true }]
}
```

Use `X-Slack-Signature` verification — never the legacy `token` field.

## Common Events and Required Scopes

### Application Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `app_home_opened` | None | User opens the App Home tab |
| `app_mention` | `app_mentions:read` | Bot is @mentioned in a channel |
| `app_uninstalled` | None | App removed from workspace — purge tokens |
| `app_rate_limited` | None | App exceeded event delivery rate (30k/hr) |

### Message Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `message.channels` | `channels:history` | Message in a public channel |
| `message.groups` | `groups:history` | Message in a private channel |
| `message.im` | `im:history` | Message in a DM |
| `message.mpim` | `mpim:history` | Message in a group DM |
| `message.app_home` | `im:history` | Message sent in App Home |

**Important**: Always filter out bot messages (`event.bot_id` present) to prevent infinite loops.

Message subtypes to be aware of:
- `bot_message` — sent by a bot
- `message_changed` — existing message was edited
- `message_deleted` — message was deleted
- `file_share` — message includes a file

### Reaction Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `reaction_added` | `reactions:read` | User added a reaction |
| `reaction_removed` | `reactions:read` | User removed a reaction |

### Channel Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `channel_created` | `channels:read` | New public channel created |
| `channel_deleted` | `channels:read` | Public channel deleted |
| `channel_rename` | `channels:read` | Public channel renamed |
| `channel_archive` | `channels:read` | Public channel archived |
| `channel_unarchive` | `channels:read` | Public channel unarchived |
| `channel_id_changed` | `channels:read` | Channel ID migrated |
| `member_joined_channel` | `channels:read` or `groups:read` | User joined a channel |
| `member_left_channel` | `channels:read` or `groups:read` | User left a channel |

### User & Team Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `team_join` | `users:read` | New member joined the workspace |
| `user_change` | `users:read` | User profile updated |
| `team_domain_change` | `team:read` | Workspace domain changed |
| `team_rename` | `team:read` | Workspace name changed |

### Token & Auth Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `tokens_revoked` | None | Bot or user tokens revoked — purge immediately |

### File Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `file_created` | `files:read` | File uploaded |
| `file_shared` | `files:read` | File shared in a channel |
| `file_deleted` | `files:read` | File deleted |
| `file_public` | `files:read` | File made public |

### Pin Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `pin_added` | `pins:read` | Item pinned in a channel |
| `pin_removed` | `pins:read` | Item unpinned |

### Link Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `link_shared` | `links:read` | A URL matching your app's domains was shared |

### Assistant Events

| Event | Scope Required | Description |
|-------|---------------|-------------|
| `assistant_thread_started` | `assistant:write` | AI assistant conversation started |
| `assistant_thread_context_changed` | `assistant:write` | Assistant thread context changed |

## Full Event Type List (120+)

**Application:** `app_deleted`, `app_home_opened`, `app_installed`, `app_mention`, `app_rate_limited`, `app_requested`, `app_uninstalled`, `app_uninstalled_team`

**Assistant:** `assistant_thread_context_changed`, `assistant_thread_started`

**Bot:** `bot_added`, `bot_changed`

**Call:** `call_rejected`

**Channel:** `channel_archive`, `channel_created`, `channel_deleted`, `channel_history_changed`, `channel_id_changed`, `channel_joined`, `channel_left`, `channel_marked`, `channel_rename`, `channel_shared`, `channel_unarchive`, `channel_unshared`

**File:** `file_change`, `file_comment_deleted`, `file_created`, `file_deleted`, `file_public`, `file_shared`, `file_unshared`

**Group:** `group_archive`, `group_close`, `group_deleted`, `group_history_changed`, `group_joined`, `group_left`, `group_marked`, `group_open`, `group_rename`, `group_unarchive`

**Message:** `message`, `message.app_home`, `message.channels`, `message.groups`, `message.im`, `message.mpim`, `message_metadata_deleted`, `message_metadata_posted`, `message_metadata_updated`

**Reaction & Pin:** `reaction_added`, `reaction_removed`, `pin_added`, `pin_removed`

**User:** `user_change`, `user_huddle_changed`, `user_typing`

**Team:** `team_access_granted`, `team_access_revoked`, `team_domain_change`, `team_join`, `team_migration_started`, `team_plan_change`, `team_rename`

**Other:** `accounts_changed`, `commands_changed`, `dnd_updated`, `dnd_updated_user`, `email_domain_changed`, `emoji_changed`, `function_executed`, `im_close`, `im_created`, `im_open`, `invite_requested`, `link_shared`, `member_joined_channel`, `member_left_channel`, `shared_channel_invite_accepted`, `shared_channel_invite_approved`, `shared_channel_invite_declined`, `shared_channel_invite_received`, `star_added`, `star_removed`, `subteam_created`, `subteam_members_changed`, `subteam_self_added`, `subteam_self_removed`, `subteam_updated`, `tokens_revoked`, `url_verification`
