# Task 1 — Step 8: Deploy to Production & Monitor

> **Estimated Duration:** 1 day (deploy + 24h monitoring)
> **Goal:** Apply all validated caching configurations to the live production server, verify everything works, and monitor for 24 hours.
> **Risk Level:** 🔴 High — mistakes here affect real users immediately.

---

## ⚠️ Read Before You Start

> This is the **most dangerous step in Task 1**. Every command you run affects the live site. Unlike staging, there's no "who cares" safety net — real users are browsing right now.
>
> **Three rules for production deployment:**
> 1. **Never type from memory** — copy the exact config that was tested on staging
> 2. **Always have a rollback command ready** before making each change
> 3. **One change at a time** — verify each change works before moving to the next

---

## 📋 Phase 1: Pre-Deployment (Before Touching Production)

### 1.1 — Confirm Staging QA Passed

Do NOT proceed unless **every item** in Step 7's checklist passed on staging:

```bash
# On staging — quick sanity check
curl -I https://staging.yourdomain.com/ | grep "X-FastCGI-Cache"
# Must show: X-FastCGI-Cache: HIT

wp redis status
# Must show: Connected, Status: enabled

wp plugin list --status=active | grep -E "wp-rocket|w3-total-cache|redis-cache|nginx-helper"
# Must show all expected plugins active
```

> 🛑 **STOP if any check fails.** Go back to Step 7 and fix staging first. Never deploy a broken config to production.

---

### 1.2 — Take Full Production Backups

```bash
# SSH into production
ssh user@production-server

# 1. Database backup (most important — this is your undo button)
wp db export ~/backups/pre-caching-deploy-$(date +%Y%m%d-%H%M%S).sql
echo "✅ Database backed up"

# 2. Verify the backup file is not empty
ls -lh ~/backups/pre-caching-deploy-*.sql
# Should be several MB at minimum — if it's 0 bytes, something went wrong

# 3. Nginx config backup
sudo cp /etc/nginx/sites-enabled/yourdomain.conf /etc/nginx/sites-enabled/yourdomain.conf.pre-caching-backup
echo "✅ Nginx config backed up"

# 4. wp-config.php backup
cp /var/www/html/wp-config.php /var/www/html/wp-config.php.pre-caching-backup
echo "✅ wp-config.php backed up"

# 5. Full files backup (if possible) or server snapshot
# Option A: Server snapshot via hosting provider (DigitalOcean, AWS, etc.)
# Option B: tar backup of wp-content
tar -czf ~/backups/wp-content-pre-caching-$(date +%Y%m%d).tar.gz /var/www/html/wp-content/
echo "✅ wp-content backed up"
```

> 💡 **Junior note — Why so many backups?**
> Different things can go wrong at different stages. The database backup lets you undo URL changes. The Nginx backup lets you restore server config. The wp-config backup lets you undo Redis settings. Each backup targets a specific rollback scenario.

---

### 1.3 — Verify Production Server Matches Staging

Production and staging servers are rarely identical. Check for these common differences:

```bash
# On PRODUCTION — run these checks:

# PHP version (must match staging)
php -v | head -1
# Compare with staging — if different, PHP-FPM socket path may differ

# PHP-FPM socket path (CRITICAL — wrong path = 502 errors)
ls /var/run/php/
# Common paths:
#   /var/run/php/php8.1-fpm.sock   (Ubuntu/Debian)
#   /var/run/php/php8.2-fpm.sock   (newer servers)
#   /var/run/php-fpm/www.sock      (CentOS/RHEL)
# ⚠️ If this differs from staging, you MUST update the Nginx config before deploying

# Nginx version
nginx -v

# Redis is installed AND running
redis-cli ping
# Must return: PONG
# If it returns "Connection refused" → Redis is not running on production
# Fix: sudo systemctl start redis-server && sudo systemctl enable redis-server

# Redis memory config
redis-cli CONFIG GET maxmemory
# Should show a value like "268435456" (256MB) — not "0" (unlimited)
# If "0": redis-cli CONFIG SET maxmemory 256mb
# Also set eviction policy:
# redis-cli CONFIG SET maxmemory-policy allkeys-lru

# WordPress root directory
wp option get siteurl
# Note the path — it may differ from staging

# Check available disk space (FastCGI cache needs space)
df -h /var/run/
# Ensure at least 1GB free for cache
```

> ⚠️ **Common gotcha:** Staging runs PHP 8.1 but production runs PHP 8.2 (or vice versa). The `fastcgi_pass` socket path in Nginx config uses the PHP version number. If they don't match, every page returns **502 Bad Gateway**.

---

### 1.4 — Schedule Deployment Window

```
Recommended: Deploy during your LOWEST traffic period
Typical: Tuesday-Thursday, 2:00 AM - 5:00 AM (your server's timezone)

Why not Monday?  → Weekend issues may still be unresolved
Why not Friday?  → If something breaks, you're debugging all weekend
Why 2-5 AM?      → Lowest traffic for most sites
```

### 1.5 — Notify Your Team

Before starting, send a message to your team/stakeholders:

```
Subject: Production Deployment — Caching Optimization
Time: [date] [time] - [estimated end time]
What: Deploying Nginx FastCGI Cache, Redis Object Cache, page caching plugin,
      browser caching headers, and minification settings.
Impact: Brief periods of increased response time during config reloads (~2-5 seconds each).
        Maintenance mode for ~5 minutes during plugin activation.
Rollback: Full rollback available within 2 minutes if issues detected.
Contact: [your name/phone] during deployment window.
```

---

## 📋 Phase 2: Deploy (On Production Server)

> From this point forward, every step is on the **production server**. Keep a second SSH session open as backup.

### 2.1 — Enable Maintenance Mode

```bash
# Put site in maintenance mode so visitors see a "briefly unavailable" page
# instead of a half-configured site
wp maintenance-mode activate

# Verify maintenance mode is active
curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/
# Should return: 503
```

> 💡 **Junior note:** Maintenance mode shows visitors a friendly "Briefly unavailable for scheduled maintenance" page. It's built into WordPress — no plugin needed. **Set a phone alarm** to remind yourself to disable it. Forgetting = site stays down.

---

### 2.2 — Deploy Nginx Configuration

```bash
# Step 1: Copy the TESTED config from staging (don't retype it!)
# Option A: SCP from staging
scp user@staging:/etc/nginx/sites-enabled/yourdomain.conf /tmp/yourdomain.conf.new

# Option B: If you have the config in version control (recommended)
# git pull and copy from repo

# Step 2: Review the diff before applying
diff /etc/nginx/sites-enabled/yourdomain.conf /tmp/yourdomain.conf.new
# ⚠️ READ THIS DIFF CAREFULLY. Look for:
#   - PHP-FPM socket path (must match production's PHP version)
#   - Server name (must be production domain, not staging)
#   - SSL certificate paths (may differ between servers)
#   - Cache directory paths
#   - Log file paths

# Step 3: Fix any staging-specific values
sudo nano /tmp/yourdomain.conf.new
# Check and update if needed:
#   fastcgi_pass unix:/var/run/php/php8.X-fpm.sock;  ← match prod PHP version
#   server_name yourdomain.com www.yourdomain.com;    ← not staging.yourdomain.com
#   ssl_certificate /path/to/prod/cert;               ← not staging cert

# Step 4: Create FastCGI cache directory with correct ownership
sudo mkdir -p /var/run/nginx-cache
sudo chown www-data:www-data /var/run/nginx-cache
# ⚠️ Without chown, Nginx can't write cache files → cache silently fails

# Step 5: Apply the config
sudo cp /tmp/yourdomain.conf.new /etc/nginx/sites-enabled/yourdomain.conf

# Step 6: TEST before reload (NEVER skip this)
sudo nginx -t
# Must show:
#   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
#   nginx: configuration file /etc/nginx/nginx.conf test is successful

# Step 7: Reload (NOT restart — reload = zero downtime)
sudo systemctl reload nginx

# Step 8: Verify Nginx is still running
sudo systemctl status nginx | head -5
# Must show: Active: active (running)
```

> 🔴 **If `nginx -t` fails:** Do NOT reload. Fix the error (the message tells you the exact file and line number). Your site is still running with the OLD config — nothing is broken yet.
>
> 🔴 **If reload causes issues:** Rollback immediately:
> ```bash
> sudo cp /etc/nginx/sites-enabled/yourdomain.conf.pre-caching-backup /etc/nginx/sites-enabled/yourdomain.conf
> sudo nginx -t && sudo systemctl reload nginx
> ```

---

### 2.3 — Deploy Nginx Gzip/Brotli Configuration

```bash
# If you modified nginx.conf (main config) for gzip/brotli on staging:
# Back up first
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.pre-caching-backup

# Apply gzip/brotli settings (copy from staging or add manually)
# Edit the http {} block in nginx.conf to add gzip settings from Step 4A

sudo nano /etc/nginx/nginx.conf
# Add gzip settings inside http {} block (see Step 4A for exact config)

# Test and reload
sudo nginx -t && sudo systemctl reload nginx
```

---

### 2.4 — Deploy wp-config.php Changes

```bash
# Edit wp-config.php on production
nano /var/www/html/wp-config.php

# Add these lines ABOVE "/* That's all, stop editing! */"
```

```php
/** Redis Object Cache Configuration */
define( 'WP_REDIS_HOST', '127.0.0.1' );
define( 'WP_REDIS_PORT', 6379 );
define( 'WP_REDIS_MAXTTL', 7200 );
define( 'WP_REDIS_PREFIX', 'yoursite_prod_' );
// ⚠️ Use a DIFFERENT prefix than staging!
// Staging: 'yoursite_staging_'  Production: 'yoursite_prod_'
// If both share the same Redis server, same prefix = data leakage between environments
define( 'WP_REDIS_GRACEFUL', true );
// CRITICAL: Without this, if Redis crashes, every page hangs for timeout duration

/** Disable WP-Cron (replaced by server cron in Step 2.6) */
define( 'DISABLE_WP_CRON', true );
```

---

### 2.5 — Install and Activate Plugins

```bash
# Install plugins (if not already on production)
wp plugin install redis-cache nginx-helper --activate

# Install your chosen page caching plugin (one of these):
wp plugin activate wp-rocket    # if already uploaded
# OR
wp plugin install w3-total-cache --activate

# Enable Redis Object Cache
wp redis enable

# Verify Redis connection
wp redis status
# Must show: Status: Connected
```

**Import plugin settings from staging** (don't reconfigure manually — too error-prone):

```
WP Rocket:
  → Staging: Settings → WP Rocket → Tools → Export Settings (downloads .json)
  → Production: Settings → WP Rocket → Tools → Import Settings (upload .json)

W3 Total Cache:
  → Staging: Performance → General → Export/Import → Export
  → Production: Performance → General → Export/Import → Import
```

> ⚠️ **After importing**, verify these settings are correct for production:
> - CDN URL (if different from staging)
> - Cache exclusion URLs
> - Minification exclusions

---

### 2.6 — Set Up Server Cron (Replace wp-cron)

```bash
# Add WordPress cron job to server crontab
# This replaces the default wp-cron that runs on every page load
sudo crontab -u www-data -e

# Add this line (runs WordPress cron every 5 minutes):
*/5 * * * * cd /var/www/html && php wp-cron.php > /dev/null 2>&1
```

```bash
# Verify cron is set
sudo crontab -u www-data -l | grep wp-cron
# Should show the line you just added
```

> 💡 **Junior note — Why replace wp-cron?**
> By default, WordPress runs scheduled tasks (cache preloading, scheduled posts, plugin updates) when a visitor loads a page. On high-traffic sites, this adds ~50-200ms to random page loads. Moving it to a server cron means it runs on a fixed schedule regardless of traffic — consistent performance for all visitors.

---

### 2.7 — Disable Maintenance Mode

```bash
# Take the site out of maintenance mode
wp maintenance-mode deactivate

# Verify the site is back
curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/
# Should return: 200

# ⚠️ SET A PHONE ALARM before enabling maintenance mode.
# If you forget to disable it, your site stays down until you remember.
```

---

## 📋 Phase 3: Verify (Immediately After Deploy)

### 3.1 — Staged Verification (Don't Check Everything At Once)

Verify in this order — if any step fails, stop and rollback that specific change:

```bash
# ── Check 1: Basic site is up ────────────────────────────────
curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/
# Must return: 200

# ── Check 2: Nginx FastCGI Cache is working ──────────────────
# First request = MISS (cache being built)
curl -I https://yourdomain.com/ 2>/dev/null | grep "X-FastCGI-Cache"
# X-FastCGI-Cache: MISS

# Second request = HIT (served from cache)
curl -I https://yourdomain.com/ 2>/dev/null | grep "X-FastCGI-Cache"
# X-FastCGI-Cache: HIT

# ── Check 3: Redis is connected ──────────────────────────────
wp redis status
# Status: Connected

# ── Check 4: Page caching plugin is working ──────────────────
curl -s https://yourdomain.com/ | tail -5
# WP Rocket: <!-- This website is like a Rocket -->
# W3TC: <!-- Performance optimized by W3 Total Cache -->

# ── Check 5: Browser caching headers ─────────────────────────
curl -I https://yourdomain.com/wp-content/themes/mytheme/style.css 2>/dev/null | grep "Cache-Control"
# Cache-Control: public, max-age=31536000, immutable

# ── Check 6: Gzip compression ────────────────────────────────
curl -I -H "Accept-Encoding: gzip" https://yourdomain.com/ 2>/dev/null | grep "Content-Encoding"
# Content-Encoding: gzip

# ── Check 7: No PHP/Nginx errors ─────────────────────────────
sudo tail -20 /var/log/nginx/error.log
# Should show no new errors since deployment

sudo tail -20 /var/www/html/wp-content/debug.log
# Should show no new fatal errors
```

---

### 3.2 — Functional Smoke Test

Open the production site in a browser (incognito mode) and manually check:

```
✅ Home page loads correctly (text, images, layout)
✅ A blog post loads with correct formatting
✅ Navigation menus work
✅ Contact form submits successfully
✅ Search works and returns results
✅ Login page works (wp-admin loads, you can log in)
✅ Admin dashboard is functional after login
✅ WooCommerce: Add to cart works
✅ WooCommerce: Cart page shows correct items (not cached from another user!)
✅ WooCommerce: Checkout page is not cached (form nonces are fresh)
✅ Mobile: site loads and looks correct on phone/tablet
✅ Cache clears on content update: edit a post → save → refresh → see changes
```

> ⚠️ **Critical WooCommerce check:** Open two different incognito windows. Add different products to cart in each. Verify each window shows its OWN cart contents, not the other's. If carts leak between sessions, your cache exclusions are broken — rollback the page caching plugin immediately.

---

## 📋 Phase 4: Warm the Cache

After verification passes, pre-build the cache so the first real visitors get fast responses:

```bash
# Option 1: WP Rocket auto-preloads (if using WP Rocket)
# Settings → WP Rocket → Preload → ensure "Activate Preloading" is checked
# It crawls your sitemap automatically — nothing to do

# Option 2: Manual warm for top pages
# Get your top 50 URLs from sitemap or analytics
wp eval "
\$sitemap = simplexml_load_file('https://yourdomain.com/sitemap_index.xml');
foreach (\$sitemap->sitemap as \$s) { echo \$s->loc . PHP_EOL; }
" | head -5
# Then fetch each URL:
curl -s https://yourdomain.com/ > /dev/null
curl -s https://yourdomain.com/about/ > /dev/null
curl -s https://yourdomain.com/blog/ > /dev/null
# ... etc

# Option 3: Automated sitemap crawler
# If you have wget:
wget --quiet --spider --recursive --level=1 --no-parent \
     https://yourdomain.com/sitemap.xml 2>/dev/null
```

---

## 📋 Phase 5: Monitor for 24 Hours

### 5.1 — Real-Time Log Monitoring (First 30 Minutes)

Stay logged in and watch logs for the first 30 minutes after deployment:

```bash
# Watch all critical logs simultaneously (use tmux or multiple SSH sessions)

# Terminal 1: Nginx errors
sudo tail -f /var/log/nginx/error.log

# Terminal 2: PHP-FPM errors
sudo tail -f /var/log/php*-fpm.log

# Terminal 3: WordPress debug log
tail -f /var/www/html/wp-content/debug.log

# Terminal 4: Redis log
sudo tail -f /var/log/redis/redis-server.log
```

### 5.2 — Periodic Checks (Every 2-4 Hours for 24 Hours)

```bash
# Quick health check script — run every few hours:
echo "=== Production Health Check $(date) ==="
echo ""

echo "1. Site status:"
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/)
[ "$HTTP_CODE" = "200" ] && echo "   ✅ Site is up (HTTP $HTTP_CODE)" || echo "   🔴 SITE DOWN (HTTP $HTTP_CODE)"

echo ""
echo "2. FastCGI Cache:"
curl -sI https://yourdomain.com/ | grep "X-FastCGI-Cache" | sed 's/^/   /'

echo ""
echo "3. Redis:"
wp redis status 2>&1 | head -3 | sed 's/^/   /'

echo ""
echo "4. Recent Nginx errors (last hour):"
sudo awk -v d="$(date -d '1 hour ago' '+%Y/%m/%d %H:%M')" '$0 >= d' /var/log/nginx/error.log | wc -l | xargs -I{} echo "   {} errors in last hour"

echo ""
echo "5. Server load:"
uptime | sed 's/^/   /'

echo ""
echo "6. Disk space:"
df -h / | tail -1 | awk '{print "   " $5 " used (" $4 " free)"}'

echo ""
echo "7. Redis memory:"
redis-cli INFO memory 2>/dev/null | grep "used_memory_human" | sed 's/^/   /'
```

### 5.3 — Performance Benchmarks (After 1 Hour)

After the cache is warm and traffic is flowing:

| Tool | Test URL | Target | How to Check |
|------|----------|--------|-------------|
| [GTmetrix](https://gtmetrix.com) | `https://yourdomain.com/` | TTFB < 200ms | Run test, check "Time to First Byte" |
| [PageSpeed Insights](https://pagespeed.web.dev) | Home + 2 inner pages | Score ≥ 90 desktop | Run test, check Performance score |
| [WebPageTest](https://webpagetest.org) | Home page | Speed Index improvement | Compare with baseline from Step 1 audit |

```bash
# Quick TTFB check from command line
curl -s -o /dev/null -w "TTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://yourdomain.com/
# TTFB should be < 0.2s (200ms) for cached pages
```

---

## 🔥 Rollback Procedures

> Keep this section bookmarked. If anything goes wrong, follow the matching rollback.

### Rollback: Nginx Config (site returning 502 / 500)

```bash
# Restore the backup config
sudo cp /etc/nginx/sites-enabled/yourdomain.conf.pre-caching-backup /etc/nginx/sites-enabled/yourdomain.conf
sudo nginx -t && sudo systemctl reload nginx
echo "✅ Nginx config rolled back"
```

### Rollback: Redis Crashes / Site Hangs

```bash
# Option 1: Disable Redis Object Cache (site falls back to default — no Redis needed)
wp redis disable
echo "✅ Redis disabled — site uses default PHP object cache"

# Option 2: Restart Redis if it crashed
sudo systemctl restart redis-server
wp redis enable
wp redis status
```

### Rollback: Page Caching Plugin Causes Issues

```bash
# Deactivate the caching plugin
wp plugin deactivate wp-rocket       # or w3-total-cache
wp cache flush
echo "✅ Page caching plugin deactivated"
```

### Rollback: Minification/JS Defer Breaks Site

```bash
# WP Rocket: Disable file optimization via WP-CLI
wp option update wp_rocket_settings --format=json '{"minify_css":false,"minify_js":false,"defer_all_js":false}'

# W3TC: Disable minification
wp w3tc option set minify.enabled false
wp cache flush

# Or the nuclear option — deactivate the plugin entirely
wp plugin deactivate wp-rocket
```

### Rollback: Everything (Full Restore)

```bash
# Nuclear rollback — restore everything to pre-deployment state
# 1. Restore Nginx config
sudo cp /etc/nginx/sites-enabled/yourdomain.conf.pre-caching-backup /etc/nginx/sites-enabled/yourdomain.conf
sudo cp /etc/nginx/nginx.conf.pre-caching-backup /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx

# 2. Restore wp-config.php
cp /var/www/html/wp-config.php.pre-caching-backup /var/www/html/wp-config.php

# 3. Deactivate all caching plugins
wp plugin deactivate wp-rocket redis-cache nginx-helper w3-total-cache --allow-root 2>/dev/null

# 4. Remove server cron
sudo crontab -u www-data -e
# Delete the wp-cron line you added

# 5. Flush everything
wp cache flush 2>/dev/null
redis-cli FLUSHDB 2>/dev/null

# 6. Verify site is back to normal
curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/
# Should return: 200

echo "✅ Full rollback complete — site is back to pre-deployment state"
```

---

## ✅ Final Validation Checklist — Production

| # | Check | How to Verify | Pass? |
|---|-------|---------------|-------|
| 1 | Site returns HTTP 200 | `curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/` | |
| 2 | TTFB < 200ms | `curl -s -o /dev/null -w "%{time_starttransfer}" ...` or GTmetrix | |
| 3 | PageSpeed ≥ 90 desktop | PageSpeed Insights | |
| 4 | FastCGI Cache: HIT | `curl -I ... \| grep X-FastCGI-Cache` | |
| 5 | Redis: Connected | `wp redis status` | |
| 6 | Page cache comment in HTML | `curl -s ... \| tail -5` (Rocket/W3TC comment) | |
| 7 | Browser cache headers correct | `curl -I .../style.css \| grep Cache-Control` | |
| 8 | Gzip active | `curl -I -H "Accept-Encoding: gzip" ... \| grep Content-Encoding` | |
| 9 | No Nginx errors (24h) | `sudo tail -100 /var/log/nginx/error.log` | |
| 10 | No PHP fatal errors (24h) | `tail -100 /var/www/html/wp-content/debug.log` | |
| 11 | Server cron running | `sudo crontab -u www-data -l \| grep wp-cron` | |
| 12 | WooCommerce cart isolation | Two incognito windows, different carts, verify no leakage | |
| 13 | Contact form works | Submit test form on production | |
| 14 | Login/admin works | Log in, verify admin dashboard loads | |
| 15 | Cache clears on post update | Edit post → save → refresh → see changes | |
| 16 | Maintenance mode OFF | Site returns 200, not 503 | |
| 17 | Backup files stored safely | `ls ~/backups/pre-caching-deploy-*` shows files | |
| 18 | Team notified of completion | Sent "deployment complete" message | |

---

## 📝 Output / Deliverable

After successful deployment, create a brief deployment report:

```markdown
# Production Deployment Report — Caching Optimization

**Date:** [date]
**Deployed by:** [your name]
**Duration:** [start time] — [end time]

## Changes Deployed
- Nginx FastCGI Cache configured and active
- Redis Object Cache enabled (256MB, allkeys-lru)
- [WP Rocket / W3 Total Cache] activated with [imported settings / manual config]
- Browser caching headers configured in Nginx
- Gzip compression enabled
- wp-cron replaced with server cron (every 5 minutes)
- Minification enabled (HTML + CSS + JS)

## Performance Results
| Metric | Before (Step 1 Audit) | After (Production) | Change |
|--------|----------------------|--------------------| -------|
| TTFB | [X]ms | [Y]ms | -[Z]% |
| PageSpeed Desktop | [X]/100 | [Y]/100 | +[Z] |
| PageSpeed Mobile | [X]/100 | [Y]/100 | +[Z] |

## Issues Encountered
- [None / describe any issues and how they were resolved]

## Status: ✅ Task 1 COMPLETE
```

---

> ⬅️ [Previous: Step 7 — Staging Testing & QA](step-07-staging-testing-qa.md) &nbsp;|&nbsp; ➡️ [Task 2: CDN Integration →](../task-2-cdn/step-01-audit-media-library.md)

