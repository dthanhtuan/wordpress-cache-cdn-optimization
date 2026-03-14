# Task 1 — Step 6: Server-Level Caching (Nginx FastCGI Cache)

> **Estimated Duration:** 1.5 days
> **Goal:** Configure Nginx to cache the full HTML output of PHP/WordPress, so repeat requests are served directly from Nginx without touching PHP or MySQL at all. This is the single biggest performance improvement you can make.

---

## ⚠️ Why This Step Matters Most

> **Junior note:** Think of caching like a restaurant. WordPress without server-level caching is like cooking every meal from scratch for every customer, even if 100 people order the same dish. Nginx FastCGI Cache is like pre-cooking popular dishes and keeping them warm — when a customer orders, you serve instantly without touching the kitchen (PHP/MySQL) at all.

| Request Type | Without FastCGI Cache | With FastCGI Cache |
|-------------|----------------------|--------------------|
| Anonymous visitor | Nginx → PHP-FPM → WordPress → MySQL → Response (~300-800ms) | Nginx → Cached HTML file → Response (~10-50ms) |
| Logged-in admin | Nginx → PHP-FPM → WordPress → MySQL → Response | Same path (cache is bypassed for logged-in users) |

FastCGI Cache stores the **complete HTML output** as a file on disk. Nginx serves it directly without ever invoking PHP. This means:
- **PHP-FPM processes stay free** for admin tasks and real dynamic requests
- **MySQL gets zero queries** for cached pages
- Your server can handle **10-50x more concurrent visitors** with the same hardware

---

## 🎯 Objectives
- Configure Nginx FastCGI Cache for WordPress
- Set correct cache bypass rules for logged-in users, POST requests, and WooCommerce pages
- Integrate automatic cache purging with the Nginx Helper plugin
- Achieve TTFB < 50ms for cached pages
- Disable wp-cron on page load and replace with server cron

---

## 🔧 Detailed Steps

### 6.1 — Define the FastCGI Cache Zone

Edit the **main Nginx config** (not the virtual host):
```bash
sudo nano /etc/nginx/nginx.conf
```

Add inside the `http {}` block (**BEFORE** any `server {}` or `include` directives):
```nginx
# ─── FastCGI Cache Zone Definition ──────────────────────────────────────────
#
# levels=1:2     → 2-level directory structure for cache files (faster lookups)
# keys_zone      → "wpcache" is the zone name, 100m = 100MB of RAM for cache keys
#                   100MB of keys ≈ ~800,000 cached pages
# max_size       → maximum disk space for cached HTML files
# inactive       → delete cached files not accessed for this duration
# use_temp_path  → write directly to cache dir (avoids unnecessary file copy)

fastcgi_cache_path /var/run/nginx-cache
    levels=1:2
    keys_zone=wpcache:100m
    max_size=1g
    inactive=24h
    use_temp_path=off;
```

Create the cache directory:
```bash
sudo mkdir -p /var/run/nginx-cache
sudo chown www-data:www-data /var/run/nginx-cache
```

> ⚠️ **RISK: Cache lost on reboot.** `/var/run/` is a tmpfs (RAM disk) on most Linux systems, so cache is cleared on reboot. This is usually fine (cache rebuilds quickly), but if you want cache to survive reboots, use `/var/cache/nginx/` instead:
> ```nginx
> fastcgi_cache_path /var/cache/nginx/wordpress levels=1:2 ...
> ```

---

### 6.2 — Create Cache Bypass Rules (map blocks)

Add **map blocks** in `nginx.conf` inside `http {}` (above the server blocks). These tell Nginx WHEN to skip the cache:

```nginx
# ─── FastCGI Cache Bypass Logic ─────────────────────────────────────────────
# $skip_cache = 1 means "DO NOT cache this request / DO NOT serve from cache"
#
# Junior note: The `map` directive is like an if-else statement evaluated once
# per request. It's much faster than `if` blocks inside `location` blocks
# (Nginx discourages `if` — search "if is evil nginx" to learn why).

# Rule 1: Skip cache for POST requests (form submissions, logins, etc.)
map $request_method $skip_cache_method {
    default 0;
    POST    1;
}

# Rule 2: Skip cache for URLs with query strings (?s=search, ?page=2, etc.)
map $query_string $skip_cache_query {
    default 0;
    "~*."   1;
}

# Rule 3: Skip cache for logged-in users and recent commenters
# WordPress sets these cookies when a user logs in or comments
map $http_cookie $skip_cache_cookie {
    default 0;
    "~*wordpress_logged_in_"      1;
    "~*wordpress_sec_"            1;
    "~*comment_author_"           1;
    "~*wp-postpass_"              1;
    "~*woocommerce_items_in_cart" 1;
    "~*woocommerce_cart_hash"     1;
}

# Rule 4: Skip cache for admin, login, and WooCommerce transactional URLs
map $request_uri $skip_cache_uri {
    default 0;
    "~*/wp-admin/"      1;
    "~*/wp-login.php"   1;
    "~*/wp-cron.php"    1;
    "~*/xmlrpc.php"     1;
    "~*/cart/"          1;
    "~*/checkout/"      1;
    "~*/my-account/"    1;
    "~*/addons/"        1;
    "~*add-to-cart"     1;
    "~*/wp-json/"       1;
}
```

> ⚠️ **Multisite note:** If running WordPress Multisite with subdomain setup (`site1.yourdomain.com`, `site2.yourdomain.com`), the default cache key already includes `$host`, so each subdomain gets its own cache. For **subdirectory** Multisite (`yourdomain.com/site1/`, `yourdomain.com/site2/`), the `$request_uri` includes the path prefix, so each sub-site is also separated. **No extra config needed** — but verify by checking cache files after setup.

---

### 6.3 — Configure the Virtual Host for FastCGI Caching

Edit your site's virtual host config:
```bash
sudo nano /etc/nginx/sites-enabled/yourdomain.conf
```

Update the full server block:

```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;
    root /var/www/html;
    index index.php index.html;

    # ... your existing SSL config ...

    # ─── Combine all skip_cache conditions ──────────────────────────────────
    set $skip_cache 0;

    if ($skip_cache_method)  { set $skip_cache 1; }
    if ($skip_cache_query)   { set $skip_cache 1; }
    if ($skip_cache_cookie)  { set $skip_cache 1; }
    if ($skip_cache_uri)     { set $skip_cache 1; }

    # ─── WordPress Pretty Permalinks ────────────────────────────────────────
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # ─── PHP Handler with FastCGI Cache ─────────────────────────────────────
    location ~ \.php$ {
        try_files $uri =404;

        # Pass to PHP-FPM (adjust socket path to your PHP version)
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # ─── FastCGI Cache Settings ─────────────────────────────────────────
        fastcgi_cache wpcache;
        fastcgi_cache_key "$scheme$request_method$host$request_uri";
        fastcgi_cache_valid 200 301 302  8h;    # Cache successful responses 8 hours
        fastcgi_cache_valid 404          1m;    # Cache 404s only 1 minute
        fastcgi_cache_min_uses 1;               # Cache on first request
        fastcgi_cache_lock on;                  # Prevents cache stampede
        fastcgi_cache_lock_timeout 5s;

        # Serve stale cache if PHP is down or slow — keeps site alive during issues
        fastcgi_cache_use_stale error timeout updating http_500 http_503;

        # ─── Cache Bypass Rules ─────────────────────────────────────────────
        fastcgi_cache_bypass $skip_cache;       # Don't SERVE from cache
        fastcgi_no_cache $skip_cache;           # Don't STORE in cache

        # ─── Debug Header — Shows HIT/MISS/BYPASS in response ──────────────
        # HIT    = served from cache (fast!)
        # MISS   = not in cache yet (first request, now being cached)
        # BYPASS = skipped by rule (logged-in user, POST, admin page)
        add_header X-FastCGI-Cache $upstream_cache_status;
    }

    # ─── Cache Purge Endpoint (for Nginx Helper plugin) ─────────────────────
    # Requires: nginx compiled with ngx_cache_purge module
    # Check: nginx -V 2>&1 | grep cache_purge
    # If not available, use "Delete local server cache files" in Nginx Helper
    location ~ /purge(/.*) {
        allow 127.0.0.1;
        deny all;
        fastcgi_cache_purge wpcache "$scheme$request_method$host$1";
    }
}
```

> ⚠️ **RISK: Wrong PHP-FPM socket path** will return 502 Bad Gateway. Check yours:
> ```bash
> ls /run/php/   # List available sockets — use the one matching your PHP version
> ```

---

### 6.4 — Test and Reload Nginx

```bash
# ALWAYS test config before reloading — a typo takes the entire site down
sudo nginx -t

# Expected output:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# If test passes, reload (not restart — reload = zero downtime)
sudo systemctl reload nginx
```

> 🔴 **RISK: Never run `sudo systemctl restart nginx` on production** unless absolutely necessary. `restart` drops all active connections. `reload` gracefully applies changes with zero downtime.

---

### 6.5 — Install Nginx Helper Plugin for Auto Cache Purging

Without this plugin, Nginx will keep serving stale cached HTML even after you update a post. Nginx Helper tells Nginx to delete the cached file when content changes.

```bash
wp plugin install nginx-helper --activate
```

Navigate to **Settings → Nginx Helper**:
```
✅ Enable Purge
Caching Method:    ✅ nginx FastCGI cache

# If ngx_cache_purge module is available:
Purge Method:      ✅ Using a purge/url

# If ngx_cache_purge is NOT available (most servers):
Purge Method:      ✅ Delete local server cache files
Cache Path:        /var/run/nginx-cache
                   (must EXACTLY match fastcgi_cache_path from Step 6.1)

Purge Conditions:
✅ Purge Homepage
✅ Purge Post/Page/Custom Post Type
✅ Purge Archives (category, tag, date)
✅ Purge RSS Feeds
```

> 💡 **Junior note:** To check if you have `ngx_cache_purge` module:
> ```bash
> nginx -V 2>&1 | grep -o cache_purge
> ```
> If it prints `cache_purge`, you have it. If nothing prints, use the "Delete local files" method.

---

### 6.6 — Disable wp-cron and Use Server Cron (Critical for Large Sites)

> **Junior note:** WordPress has a built-in task scheduler (wp-cron) that fires **on every page load**. This means a random visitor gets stuck waiting while WordPress runs background tasks. On high-traffic sites, multiple visitors can trigger the same task simultaneously, causing race conditions and unpredictable page load times.

Add to `wp-config.php`:
```php
// Disable WordPress cron triggered on page loads
define( 'DISABLE_WP_CRON', true );
```

Set up a real server cron job:
```bash
# Run WordPress cron every 5 minutes
(crontab -l 2>/dev/null; echo "*/5 * * * * cd /var/www/html && php wp-cron.php > /dev/null 2>&1") | crontab -

# Verify it's added
crontab -l
```

---

### 6.7 — Verify FastCGI Cache is Working

```bash
# Request 1 — Should be MISS (first request, cache being populated)
curl -I https://yourdomain.com/
# Look for: X-FastCGI-Cache: MISS

# Request 2 — Should be HIT (served from cache — this is the fast one!)
curl -I https://yourdomain.com/
# Look for: X-FastCGI-Cache: HIT

# Logged-in user simulation — Should be BYPASS
curl -I -H "Cookie: wordpress_logged_in_abc=admin" https://yourdomain.com/
# Look for: X-FastCGI-Cache: BYPASS

# POST request — Should be BYPASS
curl -I -X POST https://yourdomain.com/
# Look for: X-FastCGI-Cache: BYPASS

# WooCommerce cart page — Should be BYPASS
curl -I https://yourdomain.com/cart/
# Look for: X-FastCGI-Cache: BYPASS

# Check cache files exist on disk
ls -la /var/run/nginx-cache/
# Should see subdirectories with cached files inside

# Measure TTFB for cached page (target: < 50ms)
curl -o /dev/null -s -w "TTFB: %{time_starttransfer}s\n" https://yourdomain.com/
```

---

### 6.8 — Monitor Cache Performance

```bash
# Count total cached files
find /var/run/nginx-cache/ -type f | wc -l

# Check cache disk usage
du -sh /var/run/nginx-cache/

# Check hit/miss ratio from access log (after some traffic)
sudo awk '{for(i=1;i<=NF;i++) if($i ~ /X-FastCGI-Cache:/) print $(i+1)}' \
  /var/log/nginx/access.log | sort | uniq -c | sort -rn
# Target: HIT should be > 80% of total requests
```

---

### 6.9 — Emergency: How to Flush the Entire Cache

```bash
# Method 1: Delete all cache files on disk (fastest)
sudo find /var/run/nginx-cache/ -type f -delete

# Method 2: Via Nginx Helper plugin in WordPress Admin
# Settings → Nginx Helper → Purge Entire Cache

# Method 3: Via WP-CLI
wp nginx-helper purge-all
```

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Cache directory exists and is writable | `ls -la /var/run/nginx-cache/` — owned by `www-data` |
| 2 | First request = MISS, second = HIT | `curl -I` twice → check `X-FastCGI-Cache` header |
| 3 | Logged-in users bypass cache | `curl` with logged-in cookie → `X-FastCGI-Cache: BYPASS` |
| 4 | POST requests bypass cache | `curl -X POST` → BYPASS |
| 5 | `/cart/` and `/checkout/` bypass cache | `curl -I .../cart/` → BYPASS |
| 6 | Cache auto-purges on post update | Edit post → save → `curl -I` → MISS (then HIT on next) |
| 7 | Nginx Helper plugin active and configured | `wp plugin list` shows `nginx-helper` active |
| 8 | wp-cron disabled, server cron active | Check `wp-config.php` + `crontab -l` |
| 9 | TTFB < 50ms for cached pages | `curl -w "TTFB: %{time_starttransfer}s"` |
| 10 | Stale cache served when PHP is down | Stop PHP-FPM → site still loads (cached) |
| 11 | No Nginx config errors | `sudo nginx -t` returns OK |
| 12 | Cache disk usage stays within 1GB limit | `du -sh /var/run/nginx-cache/` |

---

## 📝 Output / Deliverable
- Updated `nginx.conf` with cache zone definition and bypass maps
- Updated virtual host config with FastCGI cache directives
- Nginx Helper plugin installed and configured
- Server cron job replacing wp-cron
- Verified TTFB < 50ms for anonymous visitors

---

> ⬅️ [Previous: Step 5 — Minification & Concatenation](step-05-minification-concatenation.md) &nbsp;|&nbsp; ➡️ [Next: Step 7 — Staging Testing & QA](step-07-staging-testing-qa.md)
