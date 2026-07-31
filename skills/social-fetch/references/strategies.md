# Typed social-fetch strategies

All requests use `magister_integration_read`. A `[service=name]` marker below
selects the typed Gateway service and is never part of the provider path.

## Specific social-research routes

Use the exact routes documented by `magister-social-research`:

```http
# TikTok video
GET [service=scrapecreators]/v2/tiktok/video
  QUERY url=<canonical post URL>

# Instagram post or reel
GET [service=scrapecreators]/v1/instagram/post
  QUERY url=<canonical post URL>

# LinkedIn post
GET [service=scrapecreators]/v1/linkedin/post
  QUERY url=<canonical post URL>

# YouTube video metadata or transcript
GET [service=scrapecreators]/v1/youtube/video
  QUERY url=<canonical video URL>
GET [service=scrapecreators]/v1/youtube/video/transcript
  QUERY url=<canonical video URL>

# Reddit keyword search (not an arbitrary private-thread reader)
GET [service=scrapecreators]/v1/reddit/search
  QUERY query=<keywords>
```

Do not alter version prefixes or invent an endpoint for a platform not listed
by the current `magister-social-research` skill.

## Public-page extraction fallback

For a canonical public post URL without a specific social-research route, use
the typed Firecrawl read action described by `magister-firecrawl`:

```http
POST [service=firecrawl]/v2/scrape
  JSON_BODY {"url":"<canonical public post URL>","formats":["markdown"]}
```

Treat auth walls, generic SPA shells, consent pages, and empty rendered content
as failures. Do not attempt a browser, archive, oEmbed host, or direct public
API from shell.

## Terminal behavior

Return `execution_unavailable` with the platform and attempted typed strategy
when:

- the platform has no documented typed route;
- the content is private, deleted, login-only, or blocked;
- the response cannot be tied to the requested canonical URL; or
- the user requested media bytes, full replies, or thread expansion that the
  reviewed route does not return.

Ask the user to paste/export the content or provide an attached local file.
