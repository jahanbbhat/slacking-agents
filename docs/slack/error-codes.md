# Slack API Error Codes Reference

> Written from Slack API reference — 2026-04-27

Slack API errors are returned as strings in `response.error` when `response.ok === false`. Always check `ok` before using response data.

```ts
const response = await client.chat.postMessage({ channel, text });
if (!response.ok) {
  throw new Error(`Slack API error: ${response.error}`);
}
```

---

## Authentication & Token Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `not_authed` | No token provided | Include `Authorization: Bearer xoxb-...` header |
| `invalid_auth` | Token is invalid or malformed | Re-authenticate; check token hasn't been truncated |
| `token_revoked` | Token was revoked | Re-run OAuth flow; purge stored token |
| `token_expired` | Token has expired (rare for bot tokens) | Re-run OAuth flow |
| `account_inactive` | Token belongs to a deactivated account | User must reactivate or re-install |
| `authentication_required` | Method requires authentication | Ensure token is passed |
| `team_deleted` | Workspace has been deleted | Purge installation from store |

---

## Scope / Permission Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `missing_scope` | Token lacks required scope | Re-install app with additional scopes |
| `not_allowed_token_type` | Wrong token type (bot vs user) | Use the correct token type for this method |
| `no_permission` | App lacks permission for this action | Check admin restrictions or required scope |
| `org_login_required` | Org-level auth required | Use org-level token |
| `ekm_access_denied` | Enterprise Key Management block | Contact workspace admin |

```ts
try {
  await client.reactions.add({ channel, timestamp, name: 'thumbsup' });
} catch (err) {
  if (err.data?.error === 'missing_scope') {
    // Prompt re-install with reactions:write scope
  }
}
```

---

## Channel / Conversation Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `channel_not_found` | Channel ID doesn't exist or bot can't see it | Verify channel ID; ensure bot is in channel |
| `not_in_channel` | Bot is not a member of the channel | Call `conversations.join` first (requires `channels:join`) |
| `is_archived` | Channel is archived | Cannot post; notify user |
| `channel_is_archived` | Same as above | Cannot post; notify user |
| `cant_invite_self` | Tried to invite bot to channel it's already in | Check membership before inviting |
| `restricted_action` | Admin has restricted this action in the channel | Inform user; cannot override |
| `user_is_restricted` | Guest user can't perform this action | Guest permissions limitation |

---

## Message Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `no_text` | Message has no `text` and no `blocks` | Always provide `text` as fallback |
| `msg_too_long` | Message exceeds 40,000 characters | Split into multiple messages |
| `too_many_attachments` | More than 100 attachments | Reduce attachment count; use blocks |
| `cant_update_message` | Can only update messages posted by this bot | Only update your own messages |
| `message_not_found` | The `ts` doesn't match any message | Verify `ts` value |
| `edit_window_closed` | Message edit time window has expired | Cannot update; post a follow-up instead |
| `cant_delete_message` | Bot can't delete this message | Only delete your own messages |
| `posting_to_general_channel_denied` | Admin blocked posting to #general | Use a different channel |

---

## Rate Limit Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `ratelimited` | Exceeded API rate limit | Read `Retry-After` header; wait and retry |
| `message_limit_exceeded` | Exceeded per-channel message rate (1/sec) | Implement queue with 1s delay per channel |

```ts
try {
  await client.chat.postMessage({ channel, text });
} catch (err) {
  if (err.data?.error === 'ratelimited') {
    const retryAfter = parseInt(err.headers?.['retry-after'] ?? '1');
    await new Promise(r => setTimeout(r, retryAfter * 1000));
    // retry
  }
}
```

---

## View / Modal Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `invalid_trigger_id` | `trigger_id` expired (>3s) or already used | Ack faster; open modal before any async work |
| `view_too_large` | View exceeds size limit | Reduce blocks or block content |
| `duplicate_external_id` | `external_id` already in use | Use a unique `external_id` per view |
| `hash_conflict` | `hash` mismatch in `views.update` | Re-fetch view hash; retry update |
| `not_found` | `view_id` doesn't exist | View may have been closed by user |
| `push_limit_reached` | Modal stack already has 3 views | Cannot push; use update instead |

---

## File Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `file_not_found` | File ID doesn't exist | Verify file ID |
| `file_deleted` | File was deleted | Cannot use deleted files |
| `not_allowed` | Can't share file to this channel | Check file visibility settings |
| `upload_url_expired` | Upload URL from `getUploadURLExternal` expired | Restart upload from step 1 |
| `invalid_cursor` | Pagination cursor is expired or malformed | Restart pagination from the beginning |

---

## User Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `user_not_found` | User ID doesn't exist | Verify user ID |
| `user_is_bot` | Method doesn't apply to bot users | Filter out bot users before calling |
| `user_is_ultra_restricted` | Single-channel guest; limited permissions | Handle gracefully |
| `cant_kick_self` | Can't remove yourself from a channel | Check user ID before kicking |

---

## Webhook Errors

| Error | Cause | Recovery |
|-------|-------|---------|
| `invalid_token` | Webhook URL is invalid or revoked | Re-install app to get new webhook |
| `action_prohibited` | Admin blocked the action | Inform user; cannot override |
| `no_text` | Webhook payload has no text | Always include `text` |

---

## General / Catch-All

| Error | Cause | Recovery |
|-------|-------|---------|
| `invalid_arguments` | One or more parameters are invalid | Check parameter names and types |
| `invalid_payload` | Malformed JSON in request body | Validate payload before sending |
| `request_timeout` | Slack timed out waiting for your ack | Ack within 3 seconds; move slow work async |
| `service_unavailable` | Slack is down | Retry with exponential backoff |
| `fatal_error` | Unexpected Slack-side error | Retry; report if persistent |
| `internal_error` | Same as above | Retry; report if persistent |

```ts
// Robust error handler pattern
function handleSlackError(error: any, method: string): never {
  const code = error.data?.error ?? 'unknown';
  switch (code) {
    case 'ratelimited':
      throw new RateLimitError(method, parseInt(error.headers?.['retry-after'] ?? '1'));
    case 'token_revoked':
    case 'token_expired':
      throw new AuthError(code);
    case 'missing_scope':
      throw new ScopeError(code);
    case 'channel_not_found':
    case 'not_in_channel':
      throw new ChannelError(code);
    default:
      throw new SlackApiError(method, code);
  }
}
```
