# Task 1 — Step 7: Staging Testing & QA

> **Estimated Duration:** 1.5 days
> **Goal:** Validate all caching configurations on staging before touching production. Ensure performance improvements are real and no regressions exist.

---

## 🎯 Objectives
- Measure TTFB and PageSpeed score after all caching is configured
- Validate cache invalidation works correctly
- Test all user scenarios (guest, logged-in, WooCommerce)
- Document results for comparison with baseline

---

## 🔧 Detailed Steps

### 7.1 — Run Performance Benchmarks

Run on **staging environment** and record results in a table:

| Tool | Test URL | What to Measure |
|------|----------|-----------------|
| [GTmetrix](https://gtmetrix.com) | Home page | TTFB, Fully Loaded Time, Performance Score |
| [WebPageTest](https://www.webpagetest.org) | Home page + inner page | TTFB Waterfall, First Byte, Speed Index |
| [PageSpeed Insights](https://pagespeed.web.dev) | Home page + inner page | Mobile Score, Desktop Score |

**Record your results in this table:**

| Metric | Baseline (Before) | After Caching | Improvement |
|--------|:-----------------:|:-------------:|:-----------:|
| TTFB (ms) | ___ | ___ | ___ |
| Page Load Time (s) | ___ | ___ | ___ |
| PageSpeed Desktop | ___ | ___ | ___ |
| PageSpeed Mobile | ___ | ___ | ___ |

---

### 7.2 — Test Cache Invalidation

Verify that cached pages are automatically refreshed when content changes:

```bash
# Step 1 — Confirm a page is cached
curl -I https://staging.yourdomain.com/sample-page/
# Expect: X-FastCGI-Cache: HIT (or X-Cache: HIT)

# Step 2 — Update the post via WP-CLI
wp post update <POST_ID> --post_title="Updated Title $(date)"

# Step 3 — Check the same URL again
curl -I https://staging.yourdomain.com/sample-page/
# Expect: X-FastCGI-Cache: MISS (cache was purged)

# Step 4 — Third request — should be cached again
curl -I https://staging.yourdomain.com/sample-page/
# Expect: X-FastCGI-Cache: HIT
```

---

### 7.3 — Test Logged-In User Caching Behavior

```
1. Log in to WordPress as admin
2. Visit the home page and several posts
3. Check response headers:
   - X-FastCGI-Cache: BYPASS  ✅ (should NOT be serving cached content to logged-in users)
4. Log out
5. Visit the same pages — should be CACHE HIT again
```

---

### 7.4 — Test WooCommerce (if applicable)

```
1. Add a product to the cart
2. Visit /cart/ — confirm: X-FastCGI-Cache: BYPASS
3. Proceed to /checkout/ — confirm: X-FastCGI-Cache: BYPASS
4. Visit /my-account/ — confirm: X-FastCGI-Cache: BYPASS
5. Visit a product page — confirm: X-FastCGI-Cache: HIT (product pages should be cached)
```

---

### 7.5 — Cross-Device & Cross-Browser Testing

Test the site on each of the following:

| Device / Browser | Page | Pass/Fail | Notes |
|-----------------|------|-----------|-------|
| Chrome Desktop | Home page | | |
| Firefox Desktop | Home page | | |
| Safari Desktop | Home page | | |
| Chrome Mobile (Android) | Home page | | |
| Safari Mobile (iOS) | Home page | | |
| Chrome Desktop | Blog post | | |
| Chrome Mobile | Contact page | | |

---

### 7.6 — Check Browser Console for JS Errors
```
For each tested page:
1. Open Chrome DevTools → Console tab
2. Look for any RED errors
3. Document: URL + error message
4. Fix before deploying to production
```

---

### 7.7 — Verify Compression and Headers
```bash
# Gzip test
curl -I --compressed https://staging.yourdomain.com/
# Expect: Content-Encoding: gzip

# CSS cache header
curl -I https://staging.yourdomain.com/wp-content/themes/mytheme/style.css
# Expect: Cache-Control: public, max-age=31536000

# Image cache header
curl -I https://staging.yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# Expect: Cache-Control: public, max-age=15778476
```

---

### 7.8 — Check Redis Object Cache Metrics
```bash
# Live Redis activity while browsing the site
redis-cli monitor

# Check stats
redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"

# Calculate hit rate
# Hit Rate = keyspace_hits / (keyspace_hits + keyspace_misses) × 100
# Target: > 80% hit rate
```

---

## ✅ Validation Checklist

| # | Check | How to Verify | Pass/Fail |
|---|-------|---------------|-----------|
| 1 | TTFB < 200 ms | GTmetrix / WebPageTest result | |
| 2 | PageSpeed ≥ 90/100 desktop | PageSpeed Insights | |
| 3 | PageSpeed ≥ 80/100 mobile | PageSpeed Insights | |
| 4 | Cache invalidation works | WP-CLI update → curl check | |
| 5 | Logged-in users bypass cache | Check `X-FastCGI-Cache: BYPASS` header | |
| 6 | Cart/checkout bypass cache | Check header on `/cart/` | |
| 7 | No JS errors on any page | Browser DevTools console | |
| 8 | Gzip compression active | `curl --compressed` response | |
| 9 | Redis hit rate > 80% | `redis-cli info stats` | |
| 10 | Forms work correctly | Submit test form → confirm receipt | |
| 11 | Images load correctly | Visual check across all pages | |
| 12 | Fonts load correctly | Visual check on typography | |

---

## 📝 Output / Deliverable
- `staging-qa-results.md` — completed table with all benchmark numbers and test results
- Screenshots of GTmetrix / PageSpeed results (after caching)

---

> ⬅️ [Previous: Step 6 — Server-Level Caching](step-06-server-level-caching.md) &nbsp;|&nbsp; ➡️ [Next: Step 8 — Deploy to Production](step-08-deploy-production.md)

