# WeChat

WeChat (微信) connects to your **personal WeChat account** via the ilinkai HTTP long-poll API. No WeChat client or WebSocket bridge is required on your machine.

---

## Prerequisites

- A WeChat account
- QR code login (bot token obtained via terminal scan)

---

## Step 1: First login

Run the interactive login command and scan the QR code with your WeChat app:

```bash
nanobot channels login weixin
```

The command fetches a QR code and prints it to the terminal. Open WeChat on your phone, go to **Settings > Devices > Scan**, and scan the code. Once confirmed, the token is saved to `~/.nanobot/weixin/account.json`.

To re-authenticate with a fresh QR code (force re-login):

```bash
nanobot channels login weixin --force
```

---

## Step 2: Configure `config.json`

Update `~/.nanobot/config.json` with the following section:

```json
{
  "channels": {
    "weixin": {
      "enabled": true,
      "allowFrom": ["YOUR_WECHAT_USER_ID"]
    }
  }
}
```

### Full available settings

```json
{
  "channels": {
    "weixin": {
      "enabled": true,
      "allowFrom": [],
      "base_url": "https://ilinkai.weixin.qq.com",
      "cdn_base_url": "https://novac2c.cdn.weixin.qq.com/c2c",
      "route_tag": null,
      "token": "",
      "state_dir": "",
      "poll_timeout": 35
    }
  }
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `enabled` | `false` | Whether the channel is active |
| `allowFrom` | `[]` | Whitelisted WeChat user IDs (see below) |
| `base_url` | `"https://ilinkai.weixin.qq.com"` | ilinkai API endpoint |
| `cdn_base_url` | `"https://novac2c.cdn.weixin.qq.com/c2c"` | CDN base URL for media uploads |
| `route_tag` | `null` | Route tag for ilinkai 1.0.3 compatibility; set to your ilinkai account tag |
| `token` | `""` | Manually set bot token (optional; QR login is preferred) |
| `state_dir` | `""` | State directory (default: `~/.nanobot/weixin/`) |
| `poll_timeout` | `35` | Long-poll timeout in seconds |

---

## Finding your WeChat user ID

After the first message is received, nanobot logs the sender's user ID:

```
WeChat inbound: from=wxid_xxxxxxxxxxxxxx items=1 bodyLen=12
```

Add that ID to `allowFrom` to authorize the user.

---

## Step 3: Start the gateway

```bash
nanobot gateway
```

---

## Supported message types

| Type | Direction | Notes |
|------|-----------|-------|
| Text | Both | Max 4000 characters per message |
| Image | Both | Downloaded and uploaded via CDN |
| Voice | Both | Automatic transcription via Whisper if Groq is configured |
| File | Both | Sent as a media item |
| Video | Both | Downloaded and uploaded via CDN |
| Quoted reply | Inbound | Detected and shown as `[引用: ...]` |

---

## Voice transcription

If you configure a Groq API key, WeChat voice messages are transcribed automatically:

```json
{
  "providers": {
    "groq": {
      "apiKey": "YOUR_GROQ_API_KEY"
    }
  }
}
```

---

## `route_tag` for ilinkai 1.0.3

If you are using an ilinkai account that requires a `route_tag`, set it in the config:

```json
{
  "channels": {
    "weixin": {
      "enabled": true,
      "allowFrom": ["YOUR_WECHAT_USER_ID"],
      "route_tag": "your_account_tag_here"
    }
  }
}
```

This is passed as the `SKRouteTag` header on every API request.

---

## How it works

The channel uses **HTTP long-polling** against the ilinkai API. It maintains a persistent `get_updates_buf` cursor and continuously polls `ilink/bot/getupdates`. Incoming messages are parsed from the `msgs[]` array, media items are AES-decrypted from the CDN, and outbound messages are sent via `ilink/bot/sendmessage`.

State (token, cursor, context tokens) is persisted to `~/.nanobot/weixin/account.json` so the session survives restarts.

---

## Troubleshooting

**QR code not displayed?**

Install the `qrcode` package:

```bash
pip install qrcode
```

**Session expired (-14)?**

The session may be paused for up to 1 hour. Force a re-login:

```bash
nanobot channels login weixin --force
```

**"No context_token" warning on reply?**

The user must send a message first so the channel can cache their `context_token`. Replies are only possible after at least one inbound message.

**Media upload fails?**

Ensure outbound access to `https://novac2c.cdn.weixin.qq.com` is not blocked by your network or proxy.
