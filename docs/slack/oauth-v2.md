# Slack OAuth v2 Reference

> Fetched from docs.slack.dev — 2026-04-27

## Authorization URL

Redirect users to `https://slack.com/oauth/v2/authorize` to begin the flow.
GovSlack apps use `https://slack-gov.com/oauth/v2/authorize`.

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `client_id` | Yes | Your app's client ID from App Management |
| `scope` | Yes* | Comma-separated bot token scopes |
| `user_scope` | Yes* | Comma-separated user token scopes |
| `redirect_uri` | No | Must use HTTPS; must match App Management setting |
| `state` | No | CSRF protection token — always include this |
| `team` | No | Pre-select a workspace by team ID |

*At least one of `scope` or `user_scope` is required.

### Building the Authorization URL

```ts
import crypto from 'crypto';

function buildAuthUrl(clientId: string, scopes: string[]): { url: string; state: string } {
  const state = crypto.randomBytes(16).toString('hex');
  const url = new URL('https://slack.com/oauth/v2/authorize');
  url.searchParams.set('client_id', clientId);
  url.searchParams.set('scope', scopes.join(','));
  url.searchParams.set('state', state);
  return { url: url.toString(), state };
}
```

## Callback Handling

Slack redirects to your `redirect_uri` with:
- `code` — temporary authorization code (expires in 10 minutes)
- `state` — must match what you sent; reject if it doesn't (CSRF protection)

```ts
app.get('/slack/oauth/callback', async (req, res) => {
  const { code, state } = req.query;

  // Always verify state before proceeding
  if (state !== req.session.oauthState) {
    res.status(403).send('Invalid state parameter');
    return;
  }

  const result = await slackClient.oauth.v2.access({
    client_id: process.env.SLACK_CLIENT_ID!,
    client_secret: process.env.SLACK_CLIENT_SECRET!,
    code: code as string,
  });

  // Store installation
  await installationStore.storeInstallation(result);
  res.redirect('/success');
});
```

## Token Exchange

POST to `https://slack.com/api/oauth.v2.access`.

### Response Shape

```ts
interface OAuthV2Response {
  ok: boolean;
  access_token: string;       // bot token: xoxb-...
  token_type: 'bot';
  scope: string;              // space-separated granted scopes
  bot_user_id: string;
  app_id: string;
  team: { name: string; id: string };
  enterprise: { name: string; id: string } | null;
  authed_user: {
    id: string;
    scope: string;
    access_token: string;     // user token: xoxp-...
    token_type: 'user';
  };
}
```

## Token Types

| Prefix | Type | Used for |
|--------|------|----------|
| `xoxb-` | Bot token | Most API operations, acts as the app |
| `xoxp-` | User token | Operations requiring user identity |

## Security Requirements

### State Parameter Verification
Always generate a random `state` and verify it on callback. Mismatch = reject as forgery.

```ts
// Generate at auth start
const state = crypto.randomBytes(16).toString('hex');
req.session.oauthState = state;

// Verify at callback
if (req.query.state !== req.session.oauthState) {
  throw new Error('State mismatch — possible CSRF attack');
}
```

### Signing Secret Verification
Every incoming Slack request must be verified using HMAC-SHA256.

**Headers:**
- `X-Slack-Signature` — Slack's computed signature
- `X-Slack-Request-Timestamp` — Unix timestamp of the request

```ts
import crypto from 'crypto';

function verifySlackRequest(
  signingSecret: string,
  rawBody: string,
  timestamp: string,
  signature: string
): boolean {
  // Reject requests older than 5 minutes (replay attack protection)
  const fiveMinutesAgo = Math.floor(Date.now() / 1000) - 300;
  if (parseInt(timestamp) < fiveMinutesAgo) return false;

  const baseString = `v0:${timestamp}:${rawBody}`;
  const expected = 'v0=' + crypto
    .createHmac('sha256', signingSecret)
    .update(baseString)
    .digest('hex');

  // Use timingSafeEqual to prevent timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}
```

> Note: Bolt handles this automatically when initialized with `signingSecret`. Only implement manually for custom HTTP servers.

### Token Storage

```ts
// Never store tokens in plaintext.
// Implement Bolt's InstallationStore interface:
import { Installation, InstallationStore } from '@slack/bolt';

class EncryptedInstallationStore implements InstallationStore {
  async storeInstallation(installation: Installation): Promise<void> {
    const encrypted = encrypt(JSON.stringify(installation)); // your encryption logic
    await db.installations.upsert({ teamId: installation.team?.id, data: encrypted });
  }

  async fetchInstallation(query: InstallationQuery): Promise<Installation> {
    const row = await db.installations.findOne({ teamId: query.teamId });
    return JSON.parse(decrypt(row.data));
  }

  async deleteInstallation(query: InstallationQuery): Promise<void> {
    await db.installations.delete({ teamId: query.teamId });
  }
}
```

## Token Revocation Events

```ts
// Always handle these to purge stored tokens
app.event('tokens_revoked', async ({ event }) => {
  const { oauth, bot } = event.tokens;
  for (const userId of oauth ?? []) {
    await installationStore.deleteInstallation({ userId, teamId: event.team_id });
  }
  for (const botToken of bot ?? []) {
    await installationStore.deleteInstallation({ teamId: event.team_id, isEnterpriseInstall: false });
  }
});

app.event('app_uninstalled', async ({ context }) => {
  await installationStore.deleteInstallation({ teamId: context.teamId });
});
```

## Scope Behavior
- Scopes are **additive** — re-running OAuth adds new scopes, does not replace existing ones
- Tokens cannot be downgraded without full revocation
- Handle `missing_scope` errors gracefully:

```ts
try {
  await client.someMethod({ ... });
} catch (err) {
  if (err.data?.error === 'missing_scope') {
    // Prompt user to re-install with additional scopes
  }
}
```
