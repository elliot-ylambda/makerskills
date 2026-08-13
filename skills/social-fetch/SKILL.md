---
name: social-fetch
description: "Fetch and normalize a public social post by URL (X, LinkedIn, Reddit, TikTok, HN, more) via typed Magister routes; no shell networking or user keys."
metadata:
  version: 0.2.1
---

# /social-fetch — Pull a public social post by URL

Normalize public post data for downstream research without exposing provider
credentials or giving sandboxed shell network access.

## Required typed transport

- Shell networking is unavailable. Never call a public API, provider URL,
  downloader, browser driver, SDK, or archive from shell.
- Use `magister_integration_read` only with routes documented by
  `magister-social-research`, `magister-firecrawl`, or another reviewed skill
  advertised in the current session.
- Credentials, project identity, host selection, policy, billing, and response
  bounds remain Gateway-owned. Never ask the user to add an API key to the
  machine environment.
- If no reviewed typed route represents the source, return
  `execution_unavailable` and ask the user to paste/export the post.
- Binary media download is unsupported by the JSON transport. Return media
  metadata/URLs only; do not download them.

## Step 1 — Detect platform

| URL pattern | Platform |
|---|---|
| `x.com/<user>/status/<id>` or `twitter.com/...` | X |
| `linkedin.com/posts/...` or `linkedin.com/feed/update/...` | LinkedIn |
| `instagram.com/p/...` or `instagram.com/reel/...` | Instagram |
| `tiktok.com/@.../video/...` | TikTok |
| `bsky.app/profile/.../post/...` | Bluesky |
| `reddit.com/r/.../comments/...` | Reddit |
| `<mastodon-instance>/@<user>/<id>` | Mastodon |
| `threads.net/@.../post/...` | Threads |
| `news.ycombinator.com/item?id=...` | Hacker News |
| YouTube URL | Defer to the typed social-research transcript route or request an attached file |

Reject malformed or non-public URLs. If the platform is ambiguous, ask.

## Step 2 — Select the reviewed route

Read [references/strategies.md](references/strategies.md). Prefer the most
specific typed social-research route, then the typed Firecrawl public-page
extractor. Do not invent a direct API fallback.

For each typed attempt:

1. Preserve the canonical input URL.
2. Use only the documented relative path, ordered query pairs, and bounded JSON
   body.
3. On a retryable upstream failure, retry once according to the skill policy.
4. On an auth wall, private/deleted content, unsupported route, or exhausted
   attempts, return a precise unavailable result.

## Step 3 — Normalize

Return the shape in [references/output-schema.md](references/output-schema.md):

```json
{
  "platform": "linkedin",
  "url": "https://www.linkedin.com/posts/example",
  "fetched_at": "2026-06-17T14:35:00Z",
  "raw_source": "scrapecreators",
  "author": {"handle": null, "name": "Example", "verified": null},
  "posted_at": null,
  "text": "Post text",
  "media": [],
  "engagement": {
    "likes": null,
    "reposts": null,
    "replies": null,
    "bookmarks": null,
    "views": null
  },
  "is_thread": false,
  "thread": [],
  "replies": []
}
```

Use `null`, not `0`, for unavailable metrics. Do not include the raw provider
response, credentials, or hidden/private data.

## Optional depth

- Replies/thread depth is supported only when the exact reviewed route returns
  it; never fan out to unreviewed endpoints.
- Media metadata may be returned as untrusted external provenance. Media bytes
  require a future typed asset transport and otherwise return
  `unsupported_operation`.
- Local caching is optional for normalized non-sensitive output, but it must
  not contain tokens or provider responses and must never substitute for a
  fresh result when recency matters.

## Known limits

- LinkedIn, X, Instagram, TikTok, and Threads may be auth-walled or
  anti-automation protected.
- Private/deleted posts are a hard stop.
- A generic public endpoint is not authorization to bypass the typed transport.
- Do not report a transport failure as a connection problem for company-paid
  social-research services.

## Composes with

- `deep-research`, `jab-hook`, and `business-brainstorm` for cited evidence.
- `second-brain` for a normalized text capture.
- `watch-video` for local/attached video analysis.
