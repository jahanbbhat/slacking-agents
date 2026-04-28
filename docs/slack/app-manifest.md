# Slack App Manifest Reference

> Fetched from docs.slack.dev — 2026-04-27

The app manifest is a JSON (or YAML) file that defines your entire Slack app configuration. Use it to version-control your app settings and automate app creation/updates.

## Top-Level Structure

```ts
{
  _metadata: { major_version: 2; minor_version: 1 };
  display_information: { ... };
  settings: { ... };
  features: { ... };
  oauth_config: { ... };
  outgoing_domains?: string[];
}
```

## display_information

```ts
{
  display_information: {
    name: string;             // max 35 chars; required
    description?: string;     // max 140 chars; shown in App Directory
    long_description?: string; // max 4000 chars; min 174 for public apps
    background_color?: string; // "#RRGGBB" or "#RGB"
  }
}
```

## settings

```ts
{
  settings: {
    org_deploy_enabled?: boolean;       // Enterprise Grid org-wide install
    socket_mode_enabled?: boolean;      // use Socket Mode instead of HTTP
    token_rotation_enabled?: boolean;   // enable token rotation
    is_hosted?: boolean;                // Slack-hosted app
    function_runtime?: 'remote' | 'slack'; // workflow function runtime
    allowed_ip_address_ranges?: string[]; // allowlist for API calls

    event_subscriptions?: {
      request_url?: string;             // omit for Socket Mode
      bot_events?: string[];            // e.g. ["app_mention", "message.im"]
      user_events?: string[];
    };

    interactivity?: {
      is_enabled: boolean;
      request_url?: string;             // omit for Socket Mode
      message_menu_options_url?: string; // for external_select elements
    };

    incoming_webhooks?: {
      incoming_webhooks_enabled: boolean;
    };
  }
}
```

## features

```ts
{
  features: {
    app_home?: {
      home_tab_enabled?: boolean;       // show the Home tab
      messages_tab_enabled?: boolean;   // show the Messages tab
      messages_tab_read_only_enabled?: boolean;
    };

    bot_user?: {
      display_name: string;             // max 80 chars
      always_online?: boolean;          // show as always active
    };

    shortcuts?: Array<{
      name: string;                     // max 25 chars
      callback_id: string;              // max 255 chars
      description: string;              // max 150 chars
      type: 'global' | 'message';
    }>;                                 // max 10 shortcuts

    slash_commands?: Array<{
      command: string;                  // must start with /; max 32 chars
      description: string;              // max 2000 chars
      usage_hint?: string;              // max 1000 chars; shown as placeholder
      url?: string;                     // omit for Socket Mode
      should_escape?: boolean;          // convert @mentions and #channels in text
    }>;                                 // max 50 commands

    unfurl_domains?: string[];          // max 5 domains; enables link_shared event
  }
}
```

## oauth_config

```ts
{
  oauth_config: {
    scopes: {
      bot?: string[];           // bot token scopes
      user?: string[];          // user token scopes
      bot_optional?: string[];  // user can deny these at install
      user_optional?: string[];
    };
    redirect_urls?: string[];   // must use HTTPS
    token_management_enabled?: boolean;
  }
}
```

## Complete Example

```json
{
  "_metadata": {
    "major_version": 2,
    "minor_version": 1
  },
  "display_information": {
    "name": "My Bot",
    "description": "Helps teams track tasks",
    "background_color": "#2c2d30"
  },
  "settings": {
    "org_deploy_enabled": false,
    "socket_mode_enabled": false,
    "token_rotation_enabled": false,
    "event_subscriptions": {
      "request_url": "https://your-app.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://your-app.com/slack/actions"
    },
    "incoming_webhooks": {
      "incoming_webhooks_enabled": false
    }
  },
  "features": {
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": false
    },
    "bot_user": {
      "display_name": "My Bot",
      "always_online": false
    },
    "shortcuts": [
      {
        "name": "Create Task",
        "callback_id": "create_task_shortcut",
        "description": "Quickly create a new task",
        "type": "global"
      },
      {
        "name": "Save Message",
        "callback_id": "save_message_shortcut",
        "description": "Save this message as a task",
        "type": "message"
      }
    ],
    "slash_commands": [
      {
        "command": "/task",
        "description": "Manage your tasks",
        "usage_hint": "/task [add|list|done] [text]",
        "url": "https://your-app.com/slack/commands",
        "should_escape": true
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "chat:write",
        "commands",
        "im:history",
        "im:write",
        "users:read"
      ]
    },
    "redirect_urls": ["https://your-app.com/slack/oauth/callback"]
  },
  "outgoing_domains": ["api.your-service.com"]
}
```

## Managing Manifests via API

```ts
// Create an app from a manifest (requires apps:write scope on a config token)
const result = await client.apps.manifest.create({
  manifest: JSON.stringify(manifest),
});

// Update an existing app's manifest
await client.apps.manifest.update({
  app_id: 'A0APPID',
  manifest: JSON.stringify(updatedManifest),
});

// Export the current manifest
const { manifest } = await client.apps.manifest.export({ app_id: 'A0APPID' });
```

## Socket Mode vs HTTP

| Setting | HTTP Mode | Socket Mode |
|---------|-----------|-------------|
| `socket_mode_enabled` | `false` | `true` |
| `event_subscriptions.request_url` | Required | Omit |
| `interactivity.request_url` | Required | Omit |
| `slash_commands[].url` | Required | Omit |
| Public URL required | Yes | No (good for local dev) |
