# NollsNews digest

A Cloudflare Worker that posts a twice-daily tech news digest to a Telegram group.

Sources (public feeds only):
- The Verge RSS
- Hacker News stories with 300+ comments (Algolia API)
- CNBC Technology RSS
- TLDR Tech latest edition

What it does:
- Canonicalizes links and strips tracking parameters
- Dedupes and groups near-duplicate headlines
- Uses the article's own `og:image` when one exists, otherwise sends text only
- Runs on an hourly cron and only posts at 09:00 and 15:00 America/Los_Angeles (DST-safe), with a per-slot lock in KV
- Answers a read-only `/status` command in the configured chat

## Configuration

Everything sensitive lives in Cloudflare encrypted secrets. The Worker refuses to run if any are missing.

| Name | Type | Purpose |
| --- | --- | --- |
| `TELEGRAM_BOT_TOKEN` | secret | Bot API token |
| `TELEGRAM_CHAT_ID` | secret | Target chat |
| `WEBHOOK_SECRET` | secret | Verified via `X-Telegram-Bot-Api-Secret-Token` |
| `ENABLE_POSTS` | var | `true` to publish |
| `NOLLSNEWS_STATE` | KV binding | Seen links and slot locks |

Copy `wrangler.toml.example` to `wrangler.toml`, fill in your KV namespace ID, then:

```
wrangler secret put TELEGRAM_BOT_TOKEN
wrangler secret put TELEGRAM_CHAT_ID
wrangler secret put WEBHOOK_SECRET
wrangler deploy
```

After deploying, register the webhook by sending `POST /admin/set-webhook` with header `X-Admin-Secret: <WEBHOOK_SECRET>`. `POST /admin/webhook-info` (same header) returns non-sensitive webhook status. Both return 403 without the secret.
