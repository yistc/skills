---
name: yistc-nuxt
description: Instructions and examples for using the Nuxt framework.
alwaysApply: true
---

# Nuxt
## References
You should fetch the latest documentation before making changes.
- antfu's Nuxt skill https://github.com/antfu/skills/blob/main/skills/nuxt/SKILL.md
- Nuxt's Official documentation https://nuxt.com/llms.txt

## Backend
## Preferences
### Cache and Route Rules
For pages that won't change frequently, we set cache headers to improve performance and let CDN cache the content. For example, for the refund policy page, we can set the following headers:
```ts
'/refund-policy': {
  headers: {
    'Cache-Control': 'public, max-age=3600, s-maxage=3600, stale-while-revalidate=3600',
  },
},
```

For nuxt js files, we set the following headers to allow caching for a longer time since they are immutable and won't change after deployment:
```ts
// default by nuxt: public, max-age=31536000, immutable
'/_nuxt/**': {
  headers: {
    'Cache-Control': 'public, max-age=3600, s-maxage=3600, stale-while-revalidate=3600, immutable',
  },
},
```
