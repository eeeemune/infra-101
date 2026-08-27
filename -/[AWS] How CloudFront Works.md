# 💚 How CloudFront Works

## 💛 What is it?
**CloudFront** is AWS's **CDN (Content Delivery Network)**.
It caches your content at edge locations around the world so a user is served from a nearby server instead of your origin far away.
Plain version: it is a global network of caches sitting in front of your origin (an S3 bucket, a load balancer, any HTTP server). A user in Seoul hits a Seoul edge, not your `us-east-1` server across the ocean.
## 💛 Why do we need it?
Serving everything straight from one origin is slow for distant users and hammers that origin. CloudFront fixes both:
- **Lower latency.** Content is served from an edge physically close to the user.
- **Offloads the origin.** A cache hit never touches your origin, so less load and less origin bandwidth cost.
- **TLS and modern protocols at the edge.** HTTPS termination, HTTP/2 and HTTP/3, close to the user.
- **Security built in.** AWS Shield absorbs DDoS, WAF can filter requests, and signed URLs or cookies gate private content.
### 🤍 Real-world use case
Your single-page app's JS, CSS, and images live in an S3 bucket in one region. Put CloudFront in front, and a user anywhere loads those assets from their nearest edge in milliseconds, while S3 only gets hit on the rare cache miss.
## 💛 How does it work?
The pieces:
- **Edge locations**: hundreds of points of presence worldwide that hold the cache and serve users.
- **Regional edge caches**: a larger mid-tier cache between the edges and your origin, so a miss at one edge can still be a hit here instead of going all the way to origin.
- **Origin**: where the real content lives (S3, an ALB, EC2, or any HTTP server).
- **Distribution**: your CloudFront config. It gets a domain like `d123.cloudfront.net` and holds your behaviors.
- **Cache behaviors**: path patterns (`/images/*`) that route to an origin with their own cache rules.
- **Cache key**: what identifies a cached object (host, path, and the specific query strings, headers, or cookies you choose to include).
- **TTL**: how long an object stays cached, driven by the origin's `Cache-Control` header or the distribution's min / default / max TTL.
### 🤍 Request Flow (hit vs miss)
```javascript
User (Seoul)
  |
  v
Nearest edge location
  |
  +-- object cached and fresh?  (cache HIT)  -> return immediately
  |
  +-- MISS -> Regional Edge Cache
                 |
                 +-- HIT  -> return, and cache at the edge
                 |
                 +-- MISS -> Origin (S3 / ALB)
                                |
                                v
                        fetch, cache along the way, return
```
The first user in a region pays the miss (a fetch back toward origin). Everyone after them, until the TTL expires, gets a fast hit.
### 🤍 Example: the origin controls TTL
The origin sends a caching header:
```javascript
Cache-Control: public, max-age=86400
```
CloudFront caches that object for 24 hours. With no header, it falls back to the distribution's default TTL.
### 🤍 Example: invalidate cached content
Force edges to drop an object before its TTL expires:
```bash
aws cloudfront create-invalidation \
  --distribution-id E123ABC \
  --paths "/index.html" "/css/*"
```
### 🤍 Example: private S3 origin with OAC
The recommended S3 setup keeps the bucket **private** and lets only CloudFront read it, through **Origin Access Control (OAC)**. Users cannot bypass CloudFront and hit S3 directly.
## 💛 Gotcha
- **Prefer versioned filenames over invalidations.** Invalidations are slow and cost money past a small free tier. If each deploy ships `app.abc123.js` (a new hash), the URL changes, so caches never serve stale files and you never invalidate.
- **The cache key is everything.** By default CloudFront may ignore query strings, cookies, and most headers. If your content varies by them and they are not in the cache key, users get the wrong cached version. Add exactly what matters, no more.
- **Caching is per-edge.** An object warm in Seoul is cold in Frankfurt. Every region's first user pays the miss.
- **Do not cache personalized responses by accident.** Authenticated or per-user content needs the right headers forwarded, or a `no-store` / `no-cache` policy, or you will serve one user's data to another.
- **Custom domain needs a cert in us-east-1.** CloudFront reads its ACM certificate only from the `us-east-1` region, regardless of where your origin lives.
## 💛 References
- AWS Docs: What is CloudFront: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html
- AWS Docs: How CloudFront delivers content: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html
- AWS Docs: Invalidating files: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
