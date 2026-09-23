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
- Scannable posts: bold headline, a 2-3 sentence summary, and the source link
- Summaries come from Workers AI (free allocation) reading the article text, the feed text and page metadata; when the article is paywalled or blocked it uses what is freely available (feed text, page description, and for Hacker News stories the top discussion comments). If nothing usable comes back it falls back to the one-line description
- Uses the article's own `og:image` as a photo post when one exists; otherwise a text post that relies on Telegram's link preview (the TLDR edition stays text-only, no preview)
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
| `AI` | Workers AI binding | Summaries (optional: without it posts use the one-line description) |

Copy `wrangler.toml.example` to `wrangler.toml`, fill in your KV namespace ID, then:

```
wrangler secret put TELEGRAM_BOT_TOKEN
wrangler secret put TELEGRAM_CHAT_ID
wrangler secret put WEBHOOK_SECRET
wrangler deploy
```

After deploying, register the webhook by sending `POST /admin/set-webhook` with header `X-Admin-Secret: <WEBHOOK_SECRET>`. `POST /admin/webhook-info` (same header) returns non-sensitive webhook status. `POST /admin/preview` (same header) posts one or two not-yet-posted stories marked "PREVIEW - new format" without touching the seen list or slot locks. All admin routes return 403 without the secret.
