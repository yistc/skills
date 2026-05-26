---
name: use-tinyfish-cli
description: Use TinyFish CLI for web search and web fetch. Use when you need to discover web resources and URLs (search), or extract clean content from web pages (fetch).
---

# TinyFish CLI

You have access to the TinyFish CLI (`tinyfish`) — specifically the **search** and **fetch** tools. It has been installed.

---

## Picking the Right Tool

```
search  →  fetch
lightest    heavier
```

| Tool | When to use | Speed | Cost |
|------|-------------|-------|------|
| **search** | You need to find URLs or discover web resources on a topic | Fastest | Lowest |
| **fetch** | You have URLs and need their clean content (articles, docs, product pages) | Fast | Low |

### Common Pattern

**Research: search → fetch**
Search for a topic, then fetch the best results to read their full content.

```bash
# 1. Find URLs
tinyfish search query "best React state management libraries 2026"

# 2. Read the top results
tinyfish fetch content get --format markdown "https://result1.com" "https://result2.com"
```

---

## Commands

### `tinyfish search query`

Web search. Returns ranked results with titles, URLs, and snippets.

```bash
tinyfish search query "<query>" [--location <hint>] [--language <hint>] [--pretty]
```

- Returns 10 results by default
- Use `--location` and `--language` for geo-targeted results
- Default output is JSON; `--pretty` for human-readable

```bash
tinyfish search query "best pho in Ho Chi Minh City" --location "Vietnam" --language "en"
```

---

### `tinyfish fetch content get`

Fetch clean, extracted content from one or more URLs. Strips ads, nav, boilerplate — returns just the content.

```bash
tinyfish fetch content get <urls...> [--format markdown|html|json] [--links] [--image-links] [--pretty]
```

- Accepts **multiple URLs** in a single call — they are fetched in parallel server-side
- `--format markdown` (default) — clean readable text
- `--format json` — structured document tree
- `--links` — include all extracted links from the page
- `--image-links` — include extracted image URLs
- Response includes: `url`, `final_url`, `title`, `language`, `author`, `published_date`, `text`, `latency_ms`

```bash
# Fetch one page as markdown
tinyfish fetch content get --format markdown "https://example.com/article"

# Fetch multiple pages with links
tinyfish fetch content get --links "https://site-a.com" "https://site-b.com" "https://site-c.com"
```

---

## General Notes

- **Match the user's lang˝uage**: Respond in whatever language the user writes in.
- All commands support `--pretty` for human-readable output. Default is JSON.
- Use `--debug` on the root command or set `TINYFISH_DEBUG=1` to log HTTP requests to stderr.
