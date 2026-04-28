# Bolt.js App Initialization Reference

> Fetched from docs.slack.dev — 2026-04-27

## Installation

```bash
npm install @slack/bolt
```

## App Constructor Options

```ts
import { App, LogLevel } from '@slack/bolt';

const app = new App({
  // Required for HTTP mode
  token: process.env.SLACK_BOT_TOKEN,          // xoxb-... bot token
  signingSecret: process.env.SLACK_SIGNING_SECRET,

  // Required for Socket Mode (replaces signingSecret for incoming events)
  socketMode: true,
  appToken: process.env.SLACK_APP_TOKEN,       // xapp-... app-level token

  // Optional
  logLevel: LogLevel.DEBUG,                    // DEBUG | INFO | WARN | ERROR (default: INFO)
  logger: customLogger,                        // custom logger implementing Logger interface
  ignoreSelf: true,                            // ignore events from this bot (default: true)
  processBeforeResponse: false,                // ack before processing; useful for FaaS (default: false)
  developerMode: false,                        // extra logging + extended error messages (default: false)
  receiver: customReceiver,                    // override the entire HTTP receiver
});
```

## HTTP Mode (Production)

Requires a publicly accessible URL. Use a reverse proxy or cloud deployment. For local dev, use ngrok or cloudflared.

```ts
import { App } from '@slack/bolt';

const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
});

(async () => {
  await app.start(process.env.PORT ?? 3000);
  console.log('⚡️ Bolt app running on port', process.env.PORT ?? 3000);
})();
```

**Default route:** Slack sends all events/actions/commands to `https://your-domain.com/slack/events`.

**Required env vars:**
```
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
PORT=3000
```

## Socket Mode (Local Dev / No Public URL)

Uses a persistent WebSocket connection — no public URL or ngrok needed. Requires an App-Level Token with `connections:write` scope.

```ts
import { App } from '@slack/bolt';

const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  socketMode: true,
  appToken: process.env.SLACK_APP_TOKEN!,
});

(async () => {
  await app.start();
  console.log('⚡️ Bolt app running in Socket Mode');
})();
```

**Required env vars:**
```
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
```

**App-Level Token setup:** In your app settings → Basic Information → App-Level Tokens → Generate Token with `connections:write` scope.

## Global Middleware

Runs on every incoming request before the handler. Must call `next()` to continue the chain.

```ts
// Logging middleware
app.use(async ({ body, next, logger }) => {
  logger.info(`[${body.type}] ${(body as any).event?.type ?? (body as any).command ?? ''}`);
  await next();
});

// Auth/feature-flag middleware
app.use(async ({ body, next, client }) => {
  const userId = (body as any).user_id ?? (body as any).user?.id;
  if (userId && !(await isUserAllowed(userId))) {
    return; // halt chain without calling next()
  }
  await next();
});

// Timing middleware
app.use(async ({ next }) => {
  const start = Date.now();
  await next();
  console.log(`Handler took ${Date.now() - start}ms`);
});
```

## Error Handling

```ts
// Global error handler — catches errors from any listener
app.error(async ({ error, body, logger }) => {
  logger.error('Unhandled error:', error);
  logger.debug('Payload that caused error:', JSON.stringify(body, null, 2));
  // Optionally: report to Sentry, Datadog, etc.
});
```

Errors thrown inside listeners will be caught here. Errors in middleware are also caught if they occur after `next()` is called.

## Custom Routes (HTTP Mode Only)

Add non-Slack HTTP endpoints to the same server:

```ts
const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  customRoutes: [
    {
      path: '/health',
      method: ['GET'],
      handler: (req, res) => {
        res.writeHead(200);
        res.end('OK');
      },
    },
  ],
});
```

## processBeforeResponse (FaaS / Serverless)

In serverless environments (AWS Lambda, Google Cloud Functions), the HTTP response must be sent before the function terminates. Set `processBeforeResponse: true` to ack immediately and run handler logic synchronously before the response is sent.

```ts
const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  processBeforeResponse: true,  // ack + process happen before HTTP response is flushed
});
```

## LogLevel Enum

```ts
import { LogLevel } from '@slack/bolt';

// LogLevel.DEBUG   — all internal framework logs
// LogLevel.INFO    — start/stop messages (default)
// LogLevel.WARN    — warnings only
// LogLevel.ERROR   — errors only
```

## HTTP Mode vs Socket Mode

| | HTTP Mode | Socket Mode |
|--|-----------|-------------|
| Public URL required | Yes | No |
| Best for | Production | Local dev |
| Firewall friendly | No | Yes |
| `app.start()` param | Port number | None |
| Required token | `signingSecret` | `appToken` |
| Latency | Slightly lower | Slightly higher |
| Rate limit | Standard | Standard |

## Multi-Workspace Setup (OAuth Installation)

For apps installed across many workspaces, omit `token` and instead provide OAuth credentials plus an `installationStore`. Bolt handles token lookup per workspace automatically at runtime.

```ts
import { App, Installation, InstallationQuery } from '@slack/bolt';

const app = new App({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  clientId: process.env.SLACK_CLIENT_ID!,
  clientSecret: process.env.SLACK_CLIENT_SECRET!,
  stateSecret: process.env.SLACK_STATE_SECRET!,   // random string; used for CSRF protection
  scopes: ['chat:write', 'commands', 'app_mentions:read'],

  installationStore: {
    storeInstallation: async (installation: Installation) => {
      // Persist keyed by team ID (and enterprise ID for Grid installs)
      const key = installation.isEnterpriseInstall
        ? installation.enterprise!.id
        : installation.team!.id;
      await db.installations.upsert({ key, data: encrypt(JSON.stringify(installation)) });
    },

    fetchInstallation: async (query: InstallationQuery<boolean>) => {
      const key = query.isEnterpriseInstall ? query.enterpriseId : query.teamId;
      const row = await db.installations.findOne({ key });
      if (!row) throw new Error(`No installation found for team ${key}`);
      return JSON.parse(decrypt(row.data)) as Installation;
    },

    deleteInstallation: async (query: InstallationQuery<boolean>) => {
      const key = query.isEnterpriseInstall ? query.enterpriseId : query.teamId;
      await db.installations.delete({ key });
    },
  },

  installerOptions: {
    authVersion: 'v2',
    installPath: '/slack/install',       // serves the "Add to Slack" button
    redirectUriPath: '/slack/oauth_redirect',
    directInstall: false,
    callbackOptions: {
      success: (_installation, _opts, _req, res) => res.send('Installed! You can close this tab.'),
      failure: (error, _opts, _req, res) => res.send(`Installation failed: ${error.message}`),
    },
  },
});
```

**Required env vars for multi-workspace:**
```
SLACK_SIGNING_SECRET=...
SLACK_CLIENT_ID=...
SLACK_CLIENT_SECRET=...
SLACK_STATE_SECRET=some-random-secret-string
```

### Installation object shape

```ts
interface Installation {
  team: { id: string; name: string } | undefined;
  enterprise: { id: string; name: string } | undefined;
  isEnterpriseInstall: boolean;
  appId: string;
  bot?: {
    token: string;   // xoxb-...
    scopes: string[];
    userId: string;
    id: string;
  };
  user: {
    token?: string;  // xoxp-... (only if user scopes requested)
    scopes?: string[];
    id: string;
  };
}
```

### Token lookup at runtime

Bolt calls `fetchInstallation` automatically before each handler using the `team_id` from the incoming payload. Inside handlers, use `context.client` — it is already configured with the correct workspace token.

```ts
app.event('app_mention', async ({ event, context }) => {
  // context.client uses the token fetched via installationStore.fetchInstallation
  await context.client.chat.postMessage({
    channel: event.channel,
    text: `Hello <@${event.user}>!`,
  });
});
```

## Minimal Working App (TypeScript)

```ts
import { App } from '@slack/bolt';

const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
});

app.message('hello', async ({ message, say }) => {
  await say(`Hey there <@${(message as any).user}>!`);
});

app.command('/ping', async ({ ack, respond }) => {
  await ack();
  await respond('Pong!');
});

app.error(async ({ error, logger }) => {
  logger.error(error);
});

(async () => {
  await app.start(3000);
  console.log('⚡️ Bolt app running');
})();
```
