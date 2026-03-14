# Task 1 — Step 4A: Browser Caching — Nginx Configuration

> **When to use this guide:** Your server runs **Nginx**. If you're on Apache, use [Step 4B](step-04b-browser-caching-apache.md) instead.
> **Prerequisites:** SSH access with sudo, Nginx installed, your site's virtual host config file
> **Config file:** `/etc/nginx/sites-enabled/yourdomain.conf` (or `/etc/nginx/conf.d/yourdomain.conf`)
> **Read first:** [Step 4 — Browser Caching Overview](step-04-browser-caching.md) for shared concepts (cache-busting, ETags vs immutable)

---

## 📌 Why Nginx for Browser Caching?

Nginx is the **recommended** web server for this guide. All subsequent steps (Step 6 — FastCGI Cache, CDN config) assume Nginx.

**Nginx advantages for caching:**
- Config is loaded **once** into memory — no per-request overhead (Apache re-reads `.htaccess` on every request)
- Serves static files **directly from disk** without spawning a process
- Native Brotli support (better compression than Gzip)
- `location` blocks give precise control over which files get which headers

**Nginx risk:**
- 🟠 A syntax error in config means `nginx -t` fails and reload is rejected (site keeps running with old config — safe)
- 🔴 If you use `systemctl restart nginx` instead of `reload`, ALL current connections are dropped (see safety rules below)

---

## 🔧 Detailed Steps

### 4A.1 — Add Browser Caching Headers for Static Assets

Edit your Nginx virtual host config:
```bash
sudo nano /etc/nginx/sites-enabled/yourdomain.conf
```

Add these `location` blocks inside your `server {}` block:

```nginx
# ═══════════════════════════════════════════════════════════════════════════════
# Browser Caching — Static Assets
# ═══════════════════════════════════════════════════════════════════════════════
# These location blocks tell Nginx to add Cache-Control headers to static files.
# The browser reads these headers and stores the files locally.
# On repeat visits, the browser uses the local copy instead of downloading again.

# ─── CSS & JavaScript — 1 year ────────────────────────────────────────────────
# Why 1 year? Because WordPress uses cache-busting (style.css?ver=1.2.0).
# When you deploy a new version, the ?ver= changes → browser downloads fresh.
# So it's safe to cache for 1 year — the URL changes when content changes.
location ~* \.(css|js)$ {
    etag off;
    # ETags disabled because `immutable` tells the browser "never revalidate."
    # Having both is contradictory — some browsers still send ETag checks,
    # wasting a round-trip for nothing.
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
    access_log off;
    # access_log off = don't log requests for static files.
    # These files are requested thousands of times — logging them fills
    # your access.log with noise and wastes disk I/O.
}

# ─── Images — 6 months ────────────────────────────────────────────────────────
# Why only 6 months (not 1 year)? Images in /uploads/ don't have ?ver= busting.
# If someone re-uploads a different image at the same URL, we need the browser
# to eventually check for updates. 6 months is a good balance.
# ETags stay ON here (Nginx default) so browsers can revalidate after expiry.
location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|avif)$ {
    expires 6M;
    add_header Cache-Control "public, max-age=15778476";
    access_log off;
}

# ─── Web Fonts — 1 year ───────────────────────────────────────────────────────
# Fonts rarely change and are version-controlled with the theme.
# CORS header is required because browsers enforce same-origin for fonts.
location ~* \.(woff|woff2|ttf|eot|otf)$ {
    etag off;
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
    add_header Access-Control-Allow-Origin "https://yourdomain.com";
    # ⚠️ IMPORTANT: Replace "https://yourdomain.com" with your actual domain.
    # If using a CDN, add both:
    #   add_header Access-Control-Allow-Origin "https://yourdomain.com";
    # Or if fonts are ONLY served from CDN:
    #   add_header Access-Control-Allow-Origin "https://cdn.yourdomain.com";
    #
    # ⚠️ Do NOT use wildcard "*" — it lets any website hotlink your fonts,
    #    wasting your bandwidth. Always restrict to your own domains.
    access_log off;
}

# ─── No .html/.htm block needed ───────────────────────────────────────────────
# WordPress uses pretty permalinks (/about/, /blog/my-post/) which don't end
# in .html — so a location matching *.html would never fire on most WP sites.
# HTML caching is handled by FastCGI Cache (Step 6) or your page caching plugin.
```

> 💡 **Junior note — What is a `location` block?**
> Nginx processes requests by matching the URL against `location` patterns. `location ~* \.(css|js)$` means "any URL ending in `.css` or `.js` (case-insensitive)." When a request matches, Nginx applies the directives inside that block (headers, expiry, etc.).

---

### 4A.2 — Configure ETag Headers

ETags are **enabled by default** in Nginx — no config needed for images.

For **CSS/JS/fonts** (immutable assets), we already disabled ETags above with `etag off;` inside each location block. This is correct because:
- `immutable` = "don't revalidate"
- `ETag` = "revalidate using this fingerprint"
- Having both contradicts — disable ETags when using `immutable`

**Verify ETags are working for images:**
```bash
# Images should have an ETag header (for revalidation after 6-month expiry)
curl -I https://yourdomain.com/wp-content/uploads/2024/01/photo.jpg
# Look for: ETag: "6578a9d3-1a2b3"

# CSS should NOT have an ETag header (immutable — no revalidation needed)
curl -I https://yourdomain.com/wp-content/themes/mytheme/style.css
# Should NOT contain ETag header
```

---

### 4A.3 — Enable Gzip Compression

Gzip compresses text-based files (HTML, CSS, JS, JSON) before sending them to the browser. The browser decompresses them instantly. Typical savings: **60-80% smaller files**.

Edit the main Nginx config:
```bash
sudo nano /etc/nginx/nginx.conf
```

Add inside the `http {}` block (NOT inside a `server {}` block — this applies to all sites):
```nginx
# ═══════════════════════════════════════════════════════════════════════════════
# Gzip Compression
# ═══════════════════════════════════════════════════════════════════════════════
gzip on;                    # Enable gzip
gzip_vary on;               # Tell proxies to cache both gzipped and non-gzipped versions
gzip_proxied any;           # Compress responses from proxied requests too (CDN, load balancer)
gzip_comp_level 6;          # Compression level (1-9). 6 = best balance of CPU vs compression
                            # Level 9 saves ~2% more but uses 3x more CPU — not worth it
gzip_min_length 1024;       # Don't compress files smaller than 1KB (overhead > savings)
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    application/rss+xml
    image/svg+xml;

# ⚠️ IMPORTANT: Do NOT add font/woff or font/woff2 to gzip_types!
# WOFF uses zlib compression internally. WOFF2 uses Brotli internally.
# Gzipping them wastes CPU with 0% size reduction — can even make files
# slightly LARGER (gzip wrapper adds bytes to already-compressed data).
```

> 💡 **Junior note — Why `gzip_comp_level 6`?**
> Think of it like packing a suitcase. Level 1 = throw stuff in quickly (fast, less compressed). Level 9 = fold everything perfectly (slow, max compressed). Level 6 is the sweet spot — saves ~75% bandwidth with reasonable CPU usage. Going to 9 saves only ~2% more but triples the CPU time per request.

---

### 4A.4 — Enable Brotli Compression (Optional, Better Than Gzip)

Brotli is a newer compression algorithm by Google. It compresses **15-25% better than Gzip** at the same CPU cost. All modern browsers support it. But it requires the `ngx_brotli` module, which isn't installed by default.

**Check if Brotli module is available:**
```bash
nginx -V 2>&1 | grep -o 'brotli'
# If it prints "brotli" → module is installed, proceed
# If nothing prints → module not installed, skip this step (Gzip is fine)
```

**If available**, add to `nginx.conf` inside `http {}`:
```nginx
# ═══════════════════════════════════════════════════════════════════════════════
# Brotli Compression (requires ngx_brotli module)
# ═══════════════════════════════════════════════════════════════════════════════
brotli on;
brotli_comp_level 6;        # Same logic as gzip — 6 is the sweet spot
brotli_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    application/rss+xml
    image/svg+xml;

# Brotli and Gzip can coexist! Nginx automatically picks the best one:
# - Browser supports Brotli → serve Brotli (smaller)
# - Browser only supports Gzip → serve Gzip (fallback)
# - Browser supports neither → serve uncompressed
```

> 💡 **Junior note:** You don't have to choose between Gzip and Brotli. Enable both! Nginx automatically uses Brotli when the browser supports it (sends `Accept-Encoding: br`) and falls back to Gzip otherwise.

---

### 4A.5 — Test and Reload Nginx

> ⚠️ **CRITICAL SAFETY RULE: Always `test` before `reload`. Never `restart`.**

```bash
# Step 1: Test the config for syntax errors (ALWAYS do this first)
sudo nginx -t
# Expected output:
#   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
#   nginx: configuration file /etc/nginx/nginx.conf test is successful

# Step 2: If test passes → reload (zero downtime)
sudo systemctl reload nginx

# ⚠️ NEVER use "restart" — it drops ALL active connections:
# sudo systemctl restart nginx   ← DON'T DO THIS
```

> 💡 **Junior note — `reload` vs `restart`:**
> - **reload** = Nginx reads the new config and gracefully applies it. Current requests finish normally. Zero downtime. ✅
> - **restart** = Nginx shuts down completely, then starts again. All active connections are dropped. Visitors see errors for a few seconds. ❌

**If `nginx -t` fails:**
```bash
# The error message tells you exactly which file and line has the problem:
#   nginx: [emerg] unexpected "}" in /etc/nginx/sites-enabled/yourdomain.conf:47
# Fix the error, then run nginx -t again. Your site is still running the OLD config — nothing is broken.
```

---

## ✅ Validation Checklist

| # | Check | Command | Expected |
|---|-------|---------|----------|
| 1 | Nginx config valid | `sudo nginx -t` | "syntax is ok" + "test is successful" |
| 2 | CSS headers correct | `curl -I https://yourdomain.com/wp-content/themes/mytheme/style.css` | `Cache-Control: public, max-age=31536000, immutable` |
| 3 | JS headers correct | `curl -I https://yourdomain.com/wp-content/themes/mytheme/app.js` | `Cache-Control: public, max-age=31536000, immutable` |
| 4 | Image headers correct | `curl -I https://yourdomain.com/wp-content/uploads/photo.jpg` | `Cache-Control: public, max-age=15778476` + `ETag:` present |
| 5 | Font headers correct | `curl -I https://yourdomain.com/wp-content/themes/mytheme/fonts/font.woff2` | `Cache-Control: public, max-age=31536000, immutable` + `Access-Control-Allow-Origin: https://yourdomain.com` |
| 6 | No ETag on CSS/JS | `curl -I ...style.css` | ETag header should be **absent** |
| 7 | ETag on images | `curl -I ...photo.jpg` | ETag header should be **present** |
| 8 | Gzip active | `curl -I -H "Accept-Encoding: gzip" https://yourdomain.com/` | `Content-Encoding: gzip` |
| 9 | Brotli active (if installed) | `curl -I -H "Accept-Encoding: br" https://yourdomain.com/` | `Content-Encoding: br` |
| 10 | WOFF2 NOT gzipped | `curl -I -H "Accept-Encoding: gzip" .../font.woff2` | `Content-Encoding` should be **absent** |
| 11 | Cache-busting works | Deploy CSS change → hard refresh | New CSS loads with new `?ver=` |
| 12 | PageSpeed score improved | Run PageSpeed Insights | "Leverage browser caching" warning gone |

---

## 🚨 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `nginx -t` fails with "unexpected }" | Missing semicolon or bracket | Check the line number in the error message |
| Headers not appearing on `curl -I` | Location block not matching | Check regex: `location ~* \.(css|js)$` — the `~*` means case-insensitive |
| `403 Forbidden` on static files | Nginx doesn't have read permission | `sudo chown -R www-data:www-data /var/www/html/wp-content/` |
| Fonts not loading (CORS error in console) | `Access-Control-Allow-Origin` not set or wrong domain | Check the font location block — domain must match exactly |
| Gzip not working | `gzip_types` missing the content type | Add the missing MIME type to `gzip_types` |
| Old CSS showing after deploy | Browser using cached version | Cache-busting not working — check `wp_enqueue_style()` version param |

---

> ⬅️ [Back to Step 4 — Browser Caching Overview](step-04-browser-caching.md) &nbsp;|&nbsp; ➡️ [Next: Step 5 — Minification & Concatenation](step-05-minification-concatenation.md)
