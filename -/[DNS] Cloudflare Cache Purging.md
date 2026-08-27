# 💚 Cloudflare Cache Purging

## 💛 What is it?
**Cache purging** (invalidation) removes cached copies from Cloudflare's edge so the next request re-fetches a fresh copy from your origin. It is how you push an update out immediately, instead of waiting for the cache TTL to expire on its own.
Plain version: your content is cached at Cloudflare's edge for speed. When you change it, the edge still holds the old version until it expires. Purging tells the edge "drop this now."
## 💛 Why do we need it?
- You deployed a change, but edges around the world still serve the stale version.
- You want long TTLs for a high hit ratio, yet still need the ability to force an update on demand.
- Purging is that on-demand force-refresh, for specific files or for everything.
### 🤍 Real-world use case
You fix a typo in `index.html` and redeploy. Without a purge, visitors keep seeing the old page until the edge TTL lapses. You purge that one URL, and the next visitor triggers a fresh fetch from origin.
## 💛 Purge methods (broad to surgical)
- **Purge Everything**: drops the entire zone's cache. Simple, but every edge now misses, so your origin gets a spike of traffic. Use sparingly.
- **Purge by URL (single file)**: the common one, available on all plans. You give exact URLs (up to 30 per API call on non-Enterprise plans). The URL must match what was cached, including scheme and host.
- **Purge by Cache-Tag** (Enterprise): your origin tags responses with a `Cache-Tag` header, then you purge every object carrying a tag in one call. Ideal for "purge everything about product 42."
- **Purge by Hostname** (Enterprise): drop all cached content for a hostname.
- **Purge by Prefix** (Enterprise): drop everything under a path, like `example.com/blog/*`.
## 💛 How it works (Instant Purge)
A purge propagates to every edge location in a few seconds. Cloudflare marks the matching objects stale everywhere, so the next visitor is a MISS and re-fetches from origin (or from an upper tier if Tiered Cache is on).
### 🤍 Request Flow
```javascript
You: purge request (URL / tag / everything)
  |
  v
Cloudflare API or dashboard
  |
  | propagate to all edge PoPs (seconds)
  v
Edge marks the object stale
  |
  v
next visitor -> MISS -> origin -> re-cache fresh copy
```
### 🤍 Example: purge specific URLs (API)
```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"files": ["https://example.com/css/app.css", "https://example.com/index.html"]}'
```
### 🤍 Example: purge everything
```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"purge_everything": true}'
```
### 🤍 Example: purge by Cache-Tag (Enterprise)
The origin tags its responses:
```javascript
Cache-Tag: product-42, category-shoes
```
Then you purge by tag:
```bash
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tags": ["product-42"]}'
```
### 🤍 Example: purge a URL that varies by header
If you cache with a custom cache key (for example varying by device), you must include those headers so the purge matches the right object:
```bash
-d '{"files": [{"url": "https://example.com/img.jpg", "headers": {"CF-Device-Type": "mobile"}}]}'
```
## 💛 Purge vs the alternatives
- **Versioned URLs**: ship `app.abc123.js` with a content hash. A new deploy means a new URL, so you never purge. Best default for static assets.
- **Development Mode**: temporarily **bypass** the cache for the whole zone for 3 hours while you test. This is a bypass, not a purge.
- **Short Edge Cache TTL**: let objects expire quickly on their own. Trades hit ratio for freshness.
## 💛 Gotcha
- **Purge Everything is heavy.** It empties the whole zone cache, so origin absorbs a miss spike. Reach for targeted purge (URL, tag, prefix) first.
- **URL purge must match exactly.** A wrong scheme (`http` vs `https`), a trailing slash, or a custom cache key that varies by header means the purge silently misses the object you meant. Include the same headers you cache on.
- **Rate limits apply.** Non-Enterprise plans cap files per call (around 30) and requests per minute. Batch and throttle bulk purges.
- **Cache-Tag needs Enterprise** and the tag must be on the **origin** response header. Cloudflare strips it before sending to the client, and there are limits on total tag header size.
- **Purge is fast, not instantaneous.** It is usually seconds globally, not zero. Do not purge and then immediately assert freshness in a tight loop.
- **Browser cache is separate.** Purging Cloudflare does nothing to the copy already in a user's browser. That is controlled by Browser Cache TTL and `Cache-Control`.
## 💛 References
- Cloudflare Docs: Purge cache: https://developers.cloudflare.com/cache/how-to/purge-cache/
- Cloudflare Docs: Purge by cache-tags: https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-tags/
- Cloudflare API: purge_cache endpoint: https://developers.cloudflare.com/api/operations/zone-purge
