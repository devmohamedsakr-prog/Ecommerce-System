# CDN Strategy Pattern

**Status:** Architecture Pattern | **Priority:** HIGH | **Impact:** 50% faster page loads, 53% fewer dropoffs

---

## Overview

Content Delivery Network (CDN) caches content at edge servers globally, reducing latency. Content served from nearest data center instead of central server. Global pages load 2-10x faster.

## Business Impact

- 53% of mobile users abandon pages that load > 3 seconds
- 100ms latency increase = 1% conversion loss
- CDN can reduce latency from 200ms → 50ms (4x faster)
- Result: 3-5% revenue increase from faster pages

## Architecture

```
User in Tokyo                 User in New York
       ↓                              ↓
  Edge Server (Tokyo)      Edge Server (New York)
  (5ms latency)            (5ms latency)
       │                              │
       └──────────────────┬───────────┘
                          ↓
                   CDN Network
                   (Cloudflare, Akamai,
                    AWS CloudFront)
                          ↓
                   Origin Server
                   (Central US)
```

## Content Types

### Static Assets (High Priority)
```
Images, CSS, JavaScript, fonts, videos
Cached at edge servers
TTL: 1 day - 1 year (rarely changes)
Cache Hit Rate: 95%+

Benefit: 50-100x faster delivery
Cost: $0.08-0.20 per GB
```

### Dynamic HTML (Medium Priority)
```
Product pages, category pages
Cached 1-5 minutes
Cache-busting on content change
Cache Hit Rate: 70-80%

Benefit: 5-10x faster, reduced origin load
Trade-off: Slight staleness possible
```

### User-Specific Content (Low Priority)
```
Shopping carts, user profiles, personalized pages
Can't cache (varies per user)
Not suited for CDN

Alternative: Edge computing (run logic at edge)
```

## CDN Configuration

### Cache Headers

```
Cache-Control: max-age=31536000, immutable
→ Cache for 1 year, never revalidate
For: Images, JS, CSS with versioned filenames

Cache-Control: max-age=300, must-revalidate
→ Cache for 5 minutes, revalidate with origin if stale
For: Product pages, semi-dynamic content

Cache-Control: no-cache, must-revalidate
→ Don't cache, always revalidate with origin
For: User-specific content, real-time data
```

### Cache Busting

```
Method 1: Versioning in filename
- asset.js → asset.v123.js
- New version = new URL = cache miss
- Old version still cached for old browsers

Method 2: Query string
- script.js?v=123 (less effective, some CDNs ignore)

Method 3: Purge via API
- CDN.purgeCache("/products/index.html")
- On publish, invalidate specific URLs
```

## Geographic Distribution

```
Global CDN has edge servers in:
- North America (10+ cities)
- Europe (15+ cities)
- Asia-Pacific (20+ cities)
- Latin America (5+ cities)
- Middle East (3+ cities)

User in Mumbai:
- Request → Nearest edge (Singapore/India)
- Latency: 10-20ms
- vs Central server latency: 150ms+
```

## Compression Strategy

```
Gzip (Text compression)
- HTML: 70% reduction
- CSS: 75% reduction
- JS: 65% reduction
- JSON: 80% reduction

Brotli (Better compression)
- HTML: 75% reduction
- CSS: 80% reduction
- JS: 70% reduction
- Slower to compress but faster to transfer

Cost/Benefit: Reduce bandwidth 70%, saves $0.10 per GB
```

## Security at Edge

```
1. WAF (Web Application Firewall)
   - Block SQL injection, XSS, DDoS
   - At edge (not origin)
   
2. Rate limiting
   - Block IPs making 1000+ requests/sec
   
3. Bot detection
   - Challenge suspicious traffic
   - Let legitimate traffic through
   
4. HTTPS everywhere
   - All requests encrypted
   - TLS termination at edge
```

## Example: Product Image CDN Strategy

```
Product images: 5-50MB
Upload by seller → origin storage
CDN serves to global users

Request flow:
1. User in Brazil requests product image
2. CDN edge in São Paulo checks cache
3. Cache hit (95%): Return 5ms
4. Cache miss (5%): Fetch from origin (200ms), cache, return

Cost optimization:
- Compress JPEG 70% smaller
- Cache 1 year (images immutable)
- Serve WebP to modern browsers (30% smaller)
- Result: 80% cost savings vs unoptimized

Performance:
- First user: 200ms
- Subsequent users same region: 5ms
- Across all regions: 95% served from edge
```

## CDN Providers

| Provider | Regions | Cost | Strength |
|----------|---------|------|----------|
| **CloudFlare** | 200+ | $0.20/GB | Free tier, DDoS |
| **Akamai** | 300+ | $0.15/GB | Enterprise, scale |
| **AWS CloudFront** | 500+ | $0.085/GB | AWS integration |
| **Fastly** | 60+ | $0.12/GB | Low latency, caching |

## Monitoring

```
Metrics:
- Cache hit ratio (target: 90%+)
- Edge latency (target: < 50ms)
- Origin latency (target: < 500ms)
- Bandwidth served from edge (target: 95%+)

Alerts:
- Cache hit ratio < 80%
- Edge latency > 100ms
- Origin request spike (cache issue)
```

---

**Pattern Version:** 1.0 | **Status:** Production Pattern | **Typical Gain:** 2-10x faster pages, 3-5% revenue lift

