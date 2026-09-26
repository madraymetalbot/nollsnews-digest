# NollsNews digest

A Cloudflare Worker that posts an hourly tech news digest to a Telegram group.

What it does:
- Sources (all free to read): Engadget, Hacker News 300+ comment threads (linked to HN discussion), 9to5Google, Ars Technica, TechCrunch, and the TLDR Tech edition
- Canonicalizes links, strips tracking parameters, dedupes, and mixes sources so one feed can't fill the digest
- Posts ONE newsletter-style message per slot: topic sections (AI, Tech, Gadgets, Security, Business, Policy, Science) with one short line and a source link per story, trimmed to fit Telegram's message limit
- Lines are written by Workers AI (free allocation) from the article text, feed text and page description; unavailable pages fall back to what is freely available (and for Hacker News stories, the top discussion comments). It also drops stories that repeat the same news. Without AI, lines fall back to the headline
- Runs on an hourly cron and posts from 05:00 through 21:00 America/Los_Angeles (DST-safe), only when new stories are available, with a per-slot lock in KV; stories are marked seen only after the digest posts
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
| `AI` | Workers AI binding | Digest lines (optional: without it lines use the headline) |

Copy `wrangler.toml.example` to `wrangler.toml`, fill in your KV namespace ID, then:

```
wrangler secret put TELEGRAM_BOT_TOKEN
wrangler secret put TELEGRAM_CHAT_ID
wrangler secret put WEBHOOK_SECRET
wrangler deploy
```

After deploying, register the webhook by sending `POST /admin/set-webhook` with header `X-Admin-Secret: <WEBHOOK_SECRET>`. `POST /admin/webhook-info` (same header) returns non-sensitive webhook status. `POST /admin/preview` (same header) posts the digest the next slot would build, marked "PREVIEW - new format", without touching the seen list or slot locks. All admin routes return 403 without the secret.
