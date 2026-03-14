# Task 2 — Step 9: Staging QA & Performance Testing

> **Estimated Duration:** 2 days
> **Goal:** Run comprehensive quality assurance on staging to validate every CDN integration requirement before deploying to production.
> **Risk Level:** ✅ Low — you're on staging, break things freely.

---

## ⚠️ Prerequisites — Complete Before Starting QA

| # | Prerequisite | Step |
|---|---|---|
| 1 | CDN Pull Zone set up, DNS working | Step 2 |
| 2 | Images optimized + WebP enabled (EWWW) | Step 3 |
| 3 | WordPress CDN plugin configured (URL rewriting active) | Step 4 |
| 4 | CDN caching & security policies set | Step 5 |
| 5 | Migration script tested (URLs rewritten in staging DB) | Step 6 |
| 6 | Monitoring plugins installed (Site Kit, Query Monitor) | Step 8 |

---

## 🎯 Objectives
- Measure image loading speed improvement (target: ≥ 50% faster)
- Verify all images load from CDN domain
- Test WebP delivery (Chrome) and JPEG/PNG fallback (Safari)
- Test lazy loading behavior (WordPress 5.5+ built-in)
- Test responsive images (`srcset`) on multiple viewports
- Verify hotlinking protection
- Verify HTTPS on all image resources
- Verify original image protection (requirement 4.2)
- Run a full broken image scan
- Verify monitoring plugins are working

---

## 🔧 Detailed Steps

### 9.1 — Performance Benchmark: Before vs After CDN

Run on **staging** (with CDN fully configured) and compare to baseline:

```bash
# Option 1: PageSpeed Insights (free, no API key needed)
# Run from browser: https://pagespeed.web.dev/
# Enter: https://staging.yourdomain.com/
# Run for both Mobile and Desktop

# Option 2: WebPageTest (free, multiple locations)
# https://www.webpagetest.org/
# Enter staging URL → run from Singapore, USA, Europe
# Compare waterfall charts for image load times

# Option 3: Command-line quick check
# Measure total page load time
curl -o /dev/null -s -w "TTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  https://staging.yourdomain.com/
```

**Targets:**

| Metric | Target | How to Measure |
|---|---|---|
| Image Load Time improvement | ≥ 50% vs baseline | GTmetrix/WebPageTest waterfall |
| LCP (Largest Contentful Paint) | < 2.5 seconds | PageSpeed Insights |
| PageSpeed Score (Mobile) | ≥ 80 | PageSpeed Insights |
| PageSpeed Score (Desktop) | ≥ 90 | PageSpeed Insights |
| CDN Cache Hit Rate | > 90% | BunnyCDN Dashboard |

---

### 9.2 — Verify All Images Load from CDN Domain

```bash
# Scrape all image src attributes from the home page
curl -s https://staging.yourdomain.com/ | \
  grep -oP 'src="[^"]*\.(jpg|jpeg|png|gif|webp|svg)[^"]*"' | \
  grep -v "cdn.yourdomain.com" | \
  head -20
# ✅ Expected: NO results (all images on CDN)
# ❌ If results appear: those images are NOT on CDN yet

# Check a blog post too
curl -s https://staging.yourdomain.com/sample-post/ | \
  grep -oP 'src="[^"]*\.(jpg|jpeg|png|gif|webp|svg)[^"]*"' | \
  grep -v "cdn.yourdomain.com"
# ✅ Expected: NO results

# Also check via Chrome DevTools:
# DevTools → Network tab → filter by "Img"
# All "Domain" values should be "cdn.yourdomain.com"
```

---

### 9.3 — Test WebP Delivery

```bash
# ── Chrome (supports WebP) ──
curl -sI \
  -H "Accept: image/avif,image/webp,image/apng,image/*,*/*;q=0.8" \
  https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# ✅ Expected:
#   content-type: image/webp
#   vary: Accept

# ── Safari / non-WebP browser ──
curl -sI https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# ✅ Expected:
#   content-type: image/jpeg   (JPEG fallback)
```

**Also test in real browsers:**
- Chrome Desktop: DevTools → Network → click image → Content-Type: `image/webp` ✅
- Safari Desktop: Web Inspector → Network → Content-Type: `image/jpeg` ✅

---

### 9.4 — Test Lazy Loading (WordPress Built-in)

```bash
# Verify loading="lazy" attribute in HTML
curl -s https://staging.yourdomain.com/long-post/ | \
  grep -o 'loading="[^"]*"' | sort | uniq -c
# ✅ Expected: mostly loading="lazy"
# The FIRST image should NOT have loading="lazy" (it's the LCP image)
```

**Manual browser test:**
1. Open a long blog post on staging
2. Chrome DevTools → Network tab → filter "Img"
3. Refresh the page — only images near viewport should load
4. Slowly scroll down — new image requests appear as you scroll ✅

---

### 9.5 — Test Responsive Images on Multiple Viewports

```bash
# Check srcset is present and uses CDN URLs
curl -s https://staging.yourdomain.com/ | \
  grep -o 'srcset="[^"]*"' | head -3
# ✅ Expected:
# srcset="https://cdn.yourdomain.com/.../image-300x200.jpg 300w,
#         https://cdn.yourdomain.com/.../image-600x400.jpg 600w,
#         https://cdn.yourdomain.com/.../image-1200x800.jpg 1200w"
```

| Viewport | Expected Behavior | How to Test |
|---|---|---|
| Mobile (375px) | Smaller image loaded (~300-600px) | Chrome DevTools → Mobile emulation → check image size in Network |
| Tablet (768px) | Medium image loaded (~600-800px) | Chrome DevTools → iPad emulation |
| Desktop (1440px) | Full size loaded (~1200-1920px) | Normal browser window |

---

### 9.6 — Verify Security (Requirement 4.2)

#### Hotlinking Protection:
```bash
# ❌ Should be BLOCKED (403) — external domain
curl -sI \
  -H "Referer: https://external-evil-site.com/" \
  https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg | head -1
# Expected: HTTP/2 403

# ✅ Should WORK (200) — your own domain
curl -sI \
  -H "Referer: https://staging.yourdomain.com/" \
  https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg | head -1
# Expected: HTTP/2 200

# ✅ Should WORK (200) — no referer (direct access for email/social sharing)
curl -sI https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg | head -1
# Expected: HTTP/2 200
```

#### Original Image Protection:
```bash
# If you configured Nginx to block original (full-size) uploads:
# Test that original file is blocked
curl -sI https://staging.yourdomain.com/wp-content/uploads/2024/01/original-photo.jpg | head -1
# Expected: HTTP/2 403 (blocked)

# Test that resized version works
curl -sI https://staging.yourdomain.com/wp-content/uploads/2024/01/original-photo-1200x800.jpg | head -1
# Expected: HTTP/2 200 (allowed)
```

#### HTTPS on All Resources:
```bash
# Check for mixed content (HTTP images on HTTPS page)
curl -s https://staging.yourdomain.com/ | \
  grep -oP 'src="http://[^"]*\.(jpg|jpeg|png|gif|webp)"'
# ✅ Expected: no results (0 HTTP image URLs)
```

#### Upload Directory Security:
```bash
# Directory browsing should be blocked
curl -sI https://staging.yourdomain.com/wp-content/uploads/ | head -1
# Expected: 403 Forbidden (not a directory listing)

# PHP files should be blocked in uploads
curl -sI https://staging.yourdomain.com/wp-content/uploads/test.php | head -1
# Expected: 403 Forbidden
```

---

### 9.7 — Verify Image Optimization (EWWW)

```bash
# Check EWWW is active on staging
wp plugin status ewww-image-optimizer --url=https://staging.yourdomain.com
# Expected: Status: Active

# Upload a test image and verify optimization
wp media import /tmp/test-3000px.jpg --title="QA Test Image"

# Check it was resized to 1920px max
wp eval '
$att = get_posts(["post_type" => "attachment", "posts_per_page" => 1, "orderby" => "date", "order" => "DESC"])[0];
$meta = wp_get_attachment_metadata($att->ID);
echo "Dimensions: {$meta["width"]}x{$meta["height"]}\n";
echo "Sizes: " . implode(", ", array_keys($meta["sizes"])) . "\n";
'
# Expected: Width ≤ 1920

# Check WebP file was generated
ls -la wp-content/uploads/$(date +%Y/%m)/test-3000px*
# Expected: .jpg AND .webp versions exist

# Check EXIF metadata was stripped (privacy requirement)
# If 'identify' (ImageMagick) is available:
identify -verbose wp-content/uploads/$(date +%Y/%m)/test-3000px*.jpg | grep -i "exif"
# Expected: no EXIF data (stripped by EWWW)
```

---

### 9.8 — Verify Monitoring Plugins (from Step 8)

```bash
# Site Kit connected?
wp plugin status google-site-kit
# Expected: Active

# Query Monitor active?
wp plugin status query-monitor
# Expected: Active

# WP Debugging active?
wp plugin status wp-debugging
# Expected: Active

# WP_DEBUG_LOG enabled?
wp eval 'echo defined("WP_DEBUG_LOG") && WP_DEBUG_LOG ? "debug.log: ENABLED" : "debug.log: DISABLED"; echo "\n";'
# Expected: ENABLED

# Check debug.log is writable
ls -la wp-content/debug.log
# Expected: file exists, writable by www-data
```

**Browser checks:**
- Visit any page → Admin toolbar should show Query Monitor data (page time, queries)
- WP Admin → Site Kit → should show connected services
- WP Admin → Tools → WP Debugging → should show debug.log contents

---

### 9.9 — Full Broken Image Scan

```bash
# Option 1: WP-CLI + Broken Link Checker plugin
wp plugin install broken-link-checker --activate
# Then: WP Admin → Tools → Broken Links → filter by "images"

# Option 2: Command-line crawler (if available)
# pip install linkchecker
linkchecker https://staging.yourdomain.com/ --check-extern --ignore-url='(?!.*\.(jpg|jpeg|png|gif|webp))' 2>/dev/null | grep "404\|broken"

# Option 3: Screaming Frog SEO Spider (desktop app)
# 1. Enter staging URL → crawl
# 2. Reports → Response Codes → filter 4xx for images
# 3. Expected: 0 broken image links
```

---

### 9.10 — Document QA Results

Fill in this table and keep it for Step 10 sign-off:

| # | Test | Result | Notes |
|---|------|--------|-------|
| 1 | Image load time improvement ≥ 50% | ✅ / ❌ | ___ % improvement |
| 2 | LCP < 2.5 seconds | ✅ / ❌ | ___ s |
| 3 | PageSpeed Mobile ≥ 80 | ✅ / ❌ | ___ /100 |
| 4 | PageSpeed Desktop ≥ 90 | ✅ / ❌ | ___ /100 |
| 5 | All images on CDN domain | ✅ / ❌ | |
| 6 | WebP served to Chrome | ✅ / ❌ | |
| 7 | JPEG fallback for Safari | ✅ / ❌ | |
| 8 | Lazy loading works | ✅ / ❌ | |
| 9 | srcset on all images | ✅ / ❌ | |
| 10 | Responsive images correct per viewport | ✅ / ❌ | |
| 11 | Hotlinking blocked (external referer → 403) | ✅ / ❌ | |
| 12 | Original images protected | ✅ / ❌ | |
| 13 | HTTPS on all resources | ✅ / ❌ | |
| 14 | No mixed content | ✅ / ❌ | |
| 15 | Upload directory browsing blocked | ✅ / ❌ | |
| 16 | PHP blocked in uploads | ✅ / ❌ | |
| 17 | EWWW optimizing + WebP generating | ✅ / ❌ | |
| 18 | EXIF metadata stripped | ✅ / ❌ | |
| 19 | Images resized to max 1920px | ✅ / ❌ | |
| 20 | 0 broken images | ✅ / ❌ | |
| 21 | Site Kit / Query Monitor working | ✅ / ❌ | |
| 22 | CDN cache hit rate > 90% | ✅ / ❌ | ___ % |
| 23 | SSL valid on CDN subdomain | ✅ / ❌ | |
| 24 | Origin still serves files (no 301) | ✅ / ❌ | |
| 25 | TLS 1.2+ enforced | ✅ / ❌ | |

> **All 25 tests must pass before proceeding to Step 10 (production deploy).**

---

> ⬅️ [Previous: Step 8 — Analytics & Monitoring](step-08-analytics-monitoring.md) &nbsp;|&nbsp; ➡️ [Next: Step 10 — Deploy to Production](step-10-deploy-production.md)

