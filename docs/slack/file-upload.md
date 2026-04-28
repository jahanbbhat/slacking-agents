# Slack File Upload API Reference

> Fetched from docs.slack.dev — 2026-04-27

Slack uses a **3-step upload flow** via the External Upload API. The old `files.upload` method is deprecated — always use the new flow.

**Required scope:** `files:write`

## The 3-Step Flow

```
1. files.getUploadURLExternal  →  get upload_url + file_id
2. PUT to upload_url           →  transfer the file bytes
3. files.completeUploadExternal →  finalize and share to channel(s)
```

## Step 1: Get Upload URL

```ts
import { WebClient } from '@slack/web-api';
import fs from 'fs';

const client = new WebClient(process.env.SLACK_BOT_TOKEN);
const filePath = './report.pdf';
const fileBuffer = fs.readFileSync(filePath);

const { upload_url, file_id } = await client.files.getUploadURLExternal({
  filename: 'report.pdf',
  length: fileBuffer.byteLength,
  // optional:
  snippet_type: 'python',           // syntax highlighting for code files
  alt_txt: 'Monthly report PDF',    // accessibility description (max 1000 chars)
});
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `filename` | Yes | Name of the file including extension |
| `length` | Yes | File size in bytes |
| `snippet_type` | No | Language for syntax highlighting (e.g. `typescript`, `json`) |
| `alt_txt` | No | Alt text for images; max 1000 chars |

### Response

```ts
{ ok: true, upload_url: 'https://files.slack.com/upload/v1/...', file_id: 'F123ABC456' }
```

## Step 2: Upload the File

PUT the raw file bytes to `upload_url`. No Slack auth header needed here — the URL is pre-signed.

```ts
const uploadResponse = await fetch(upload_url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/octet-stream' },
  body: fileBuffer,
});

if (!uploadResponse.ok) {
  throw new Error(`Upload failed: ${uploadResponse.status}`);
}
```

## Step 3: Complete the Upload

```ts
await client.files.completeUploadExternal({
  files: [
    {
      id: file_id,
      title: 'Monthly Report — April 2026',   // optional display title
    },
  ],
  channel_id: 'C0CHANNEL_ID',   // share to this channel
  initial_comment: 'Here is the April report :page_facing_up:',  // optional message
  thread_ts: '1234567890.123456',  // optional: share in a thread
});
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `files` | Yes | Array of `{ id, title? }` objects |
| `channel_id` | No | Share to this channel immediately |
| `initial_comment` | No | Message to post alongside the file |
| `thread_ts` | No | Share in a specific thread |

## Complete Helper Function

```ts
import { WebClient } from '@slack/web-api';
import fs from 'fs';
import fetch from 'node-fetch';

async function uploadFileToSlack(
  client: WebClient,
  filePath: string,
  channelId: string,
  options?: { title?: string; comment?: string; threadTs?: string }
): Promise<string> {
  const fileBuffer = fs.readFileSync(filePath);
  const filename = filePath.split('/').pop()!;

  // Step 1
  const { upload_url, file_id } = await client.files.getUploadURLExternal({
    filename,
    length: fileBuffer.byteLength,
  });

  // Step 2
  const uploadRes = await fetch(upload_url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/octet-stream' },
    body: fileBuffer,
  });
  if (!uploadRes.ok) throw new Error(`Upload failed: ${uploadRes.status}`);

  // Step 3
  await client.files.completeUploadExternal({
    files: [{ id: file_id!, title: options?.title ?? filename }],
    channel_id: channelId,
    initial_comment: options?.comment,
    thread_ts: options?.threadTs,
  });

  return file_id!;
}
```

## Uploading from a Buffer (no filesystem)

```ts
import { Readable } from 'stream';

const content = 'Hello, world!';
const buffer = Buffer.from(content, 'utf-8');

const { upload_url, file_id } = await client.files.getUploadURLExternal({
  filename: 'hello.txt',
  length: buffer.byteLength,
});

await fetch(upload_url, {
  method: 'POST',
  headers: { 'Content-Type': 'text/plain' },
  body: buffer,
});

await client.files.completeUploadExternal({
  files: [{ id: file_id! }],
  channel_id: 'C0CHANNEL_ID',
});
```

## Constraints

| Constraint | Limit |
|-----------|-------|
| Required scope | `files:write` |
| Rate limit | Tier 4 (100+ req/min) |
| Max file size | 1 GB (varies by workspace plan) |
| `alt_txt` | Max 1000 chars |
| `snippet_type` values | `abap`, `arduino`, `bash`, `c`, `cpp`, `csharp`, `css`, `d`, `dart`, `diff`, `dockerfile`, `erlang`, `fortran`, `fsharp`, `go`, `groovy`, `haskell`, `html`, `java`, `javascript`, `json`, `kotlin`, `latex`, `lisp`, `lua`, `makefile`, `markdown`, `matlab`, `mermaid`, `mumps`, `objc`, `ocaml`, `pascal`, `perl`, `php`, `plaintext`, `powershell`, `prolog`, `protobuf`, `python`, `r`, `ruby`, `rust`, `sass`, `scala`, `shell`, `smalltalk`, `sql`, `swift`, `terraform`, `toml`, `typescript`, `vb`, `vbscript`, `velocity`, `verilog`, `xml`, `yaml` |

## Why Not files.upload?

`files.upload` is deprecated. It streams all file content through Slack's API servers, causing timeouts on large files and offering no pre-signed URL security. The new 3-step flow uploads directly to Slack's storage backend.
