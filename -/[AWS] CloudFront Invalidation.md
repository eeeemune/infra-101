# 💚 CloudFront Invalidation

## 💛 What is it?
A **CloudFront invalidation** removes objects from CloudFront's edge caches **before their TTL expires**, forcing the next request to re-fetch a fresh copy from the origin. It is CloudFront's manual cache purge.
Plain version: your content is cached at the edge for hours or days. When you change it, the edge keeps serving the old copy until it expires. An invalidation tells CloudFront "drop these objects now."
## 💛 Why do we need it?
- You deployed a change, but edges around the world still serve the stale version, and the TTL is long.
- The file's URL must stay the same (like `index.html`), so versioned filenames are not an option.
- Invalidation forces the refresh on demand, without waiting out the TTL.
### 🤍 Real-world use case
You push a fix to `/index.html`. Its cache TTL is a day, so visitors would see the old page for hours. You invalidate `/index.html`, and the next visitor triggers a fresh fetch from origin.
## 💛 How it works
- You create an invalidation listing one or more **paths**.
- CloudFront marks the matching objects expired at every edge location.
- It propagates globally, usually within a few minutes.
- The next viewer is a MISS, so CloudFront re-fetches from origin and re-caches.
### 🤍 Request Flow
```javascript
aws cloudfront create-invalidation --paths "/*"
  |
  v
CloudFront control plane
  |
  | mark matching objects expired at all edges (propagate, ~minutes)
  v
Edge caches: matching objects invalidated
  |
  v
next viewer -> MISS -> origin -> re-cache fresh copy
```
### 🤍 Path rules
- **Exact**: `/css/app.css` invalidates that one object.
- **Wildcard**: `/css/*` invalidates everything under that prefix; `/*` invalidates the whole distribution.
- Paths are **case-sensitive** and must start with `/`. `/Index.html` and `/index.html` are different objects.
- **Query-string variants are separate.** Invalidating `/img.jpg` does NOT clear `/img.jpg?v=2`. Use `/img.jpg*` or list each variant.
### 🤍 Example: create and check an invalidation (CLI)
```bash
# create
aws cloudfront create-invalidation \
  --distribution-id E123ABC \
  --paths "/index.html" "/css/*"

# check its status (InProgress -> Completed)
aws cloudfront get-invalidation \
  --distribution-id E123ABC \
  --id I2J3K4EXAMPLE
```
## 💛 Cost (this is the part people miss)
- The first **1,000 paths per month** are free, across all your distributions.
- After that, each path costs about **$0.005**.
- A **wildcard counts as one path**. So `/*` is a single path, while listing 2,000 individual files is 2,000 paths (1,000 free, then 1,000 billed).
The takeaway: to clear a whole directory, `/{dir}/*` is one cheap path, not a long list of files.
## 💛 Limits
- Up to **3,000 exact paths** per invalidation request.
- Up to **15 wildcard paths** per invalidation request.
- Up to **3,000 invalidations in progress** per distribution (only **15** in progress if they contain wildcards).
Hit these and you either batch differently or, better, stop relying on invalidation (see below).
## 💛 Invalidation vs versioned URLs (prefer versioning)
- **Versioned URLs**: ship `app.abc123.js` with a content hash. A new deploy produces a new URL, so the cache never serves stale files, and you never pay for or wait on an invalidation. This is the recommended default for static assets.
- **Invalidation**: use when the URL cannot change (an HTML entry point), or as a one-off emergency fix. Not the routine deploy path.
## 💛 Gotcha
- **Only CloudFront is cleared.** Browser caches and any other downstream caches are untouched. Those are governed by `Cache-Control` and Browser Cache TTL, not invalidation.
- **Cache-key variants need their own paths.** If you cache different objects per query string or header, invalidating the bare path leaves the variants cached. Use a wildcard or list them.
- **Not instant.** It is usually a few minutes, not zero. Do not build a flow that assumes the object is gone the moment the API returns.
- **Wildcard is cheaper than a file list.** One `/dir/*` beats 500 individual paths on both cost and limits.
- **Frequent invalidation is a smell.** If every deploy invalidates, switch to hashed filenames. Invalidation is the fallback, not the mechanism.
## 💛 References
- AWS Docs: Invalidating files: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
- AWS CLI: create-invalidation: https://docs.aws.amazon.com/cli/latest/reference/cloudfront/create-invalidation.html
- AWS CloudFront pricing (invalidation): https://aws.amazon.com/cloudfront/pricing/
