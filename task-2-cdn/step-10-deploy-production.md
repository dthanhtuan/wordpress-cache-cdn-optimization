# Task 2 — Step 10: Deploy to Production & Final Sign-Off

> **Estimated Duration:** 1 day (deploy + 48h monitoring)
> **Goal:** Apply all CDN configurations to production, run the migration (Step 7), monitor for 48 hours, and deliver final sign-off.
> **Risk Level:** 🔴 High — production changes. Follow checklist exactly.

---

## ⚠️ This Step Orchestrates Previous Steps on Production

Step 10 is NOT a standalone guide — it references work from previous steps. The actual commands and scripts are documented in those steps. This is your **production deployment checklist**.

```
What was tested on staging:            What Step 10 does on production:
─────────────────────────────          ──────────────────────────────
Step 3: EWWW image optimization   →    Install + configure EWWW
Step 4: CDN plugin (URL rewriting) →   Install + configure CDN plugin
Step 7: Migration (cache warm + DB) →  Run migration script
Step 8: Monitoring plugins         →   Install Site Kit, Query Monitor, etc.
```

---

## 🚦 Pre-Deployment Gate — ALL Must Be ✅

| # | Gate | Verified? |
|---|---|---|
| 1 | **All 25 Step 9 QA tests passed** on staging | ☐ |
| 2 | CDN Pull Zone configured + DNS propagated (Step 2) | ☐ |
| 3 | CDN security policies set — HTTPS, hotlinking, TLS 1.2+ (Step 5) | ☐ |
| 4 | Migration script tested on staging — 0 errors, 0 old URLs remaining (Step 6) | ☐ |
| 5 | Production database backup taken (fresh, today) | ☐ |
| 6 | Production files backup / server snapshot taken | ☐ |
| 7 | Low-traffic maintenance window scheduled | ☐ |
| 8 | Team notified of maintenance window | ☐ |
| 9 | Rollback plan reviewed (bottom of this doc) | ☐ |

---

## 🔧 Production Deployment Steps

### 10.1 — Backup Production

```bash
ssh user@production-server
cd /var/www/html

# Verify correct server!
wp eval 'echo get_bloginfo("url") . "\n";'
# Must output: https://yourdomain.com (NOT staging!)

# Database backup
wp db export backup-prod-pre-cdn-$(date +%Y%m%d-%H%M).sql
ls -lh backup-prod-pre-cdn-*.sql

# Files backup
tar -czf /tmp/backup-uploads-$(date +%Y%m%d-%H%M).tar.gz wp-content/uploads/

# Copy backups off-server
scp backup-prod-pre-cdn-*.sql user@backup-server:/backups/
```

---

### 10.2 — Install EWWW Image Optimizer (from Step 3)

```bash
# Install server-side compression tools (if not already installed)
sudo apt-get install -y jpegoptim optipng webp

# Install EWWW plugin
wp plugin install ewww-image-optimizer --activate
# For Multisite:
# wp plugin install ewww-image-optimizer --activate-network
```

**Configure EWWW** (same settings as staging — see Step 3, §3.7):
- Compression: Pixel Perfect (Lossless)
- Remove Metadata: Yes
- WebP Conversion: Enabled
- Resize: 1920px max (backup to `big_image_size_threshold` filter)

```bash
# Verify server tools are found
wp eval '
if (function_exists("ewww_image_optimizer_tool_found")) {
    echo "EWWW tools check passed\n";
} else {
    echo "EWWW not active\n";
}
'
```

---

### 10.3 — Install CDN Plugin (from Step 4)

Install the CDN plugin that matches your caching plugin choice:

```bash
# If using WP Rocket (CDN built-in — just configure):
# WP Admin → WP Rocket → CDN tab → Enable CDN → CDN URL: https://cdn.yourdomain.com

# If using W3 Total Cache (CDN built-in):
# WP Admin → Performance → CDN → Enable → CDN type: Generic Mirror
# Configuration → CDN hostname: cdn.yourdomain.com

# If using neither — install CDN Enabler (free, lightweight):
wp plugin install cdn-enabler --activate
# WP Admin → Settings → CDN Enabler → CDN Hostname: cdn.yourdomain.com
```

**Verify URL rewriting is working:**
```bash
# Check page source — images should use CDN URL
curl -s https://yourdomain.com/ | grep -o 'src="[^"]*cdn.yourdomain.com[^"]*"' | head -5
# ✅ Expected: CDN URLs in image src attributes
```

---

### 10.4 — Install Monitoring Plugins (from Step 8)

```bash
# Site Kit by Google (GA4 + PageSpeed in WP Admin)
wp plugin install google-site-kit --activate

# Query Monitor (real-time performance debugging)
wp plugin install query-monitor --activate

# WP Debugging (PHP error log viewer in WP Admin)
wp plugin install wp-debugging --activate

# Simple History (optional — action audit log)
wp plugin install simple-history --activate
```

**Enable debug logging** (if not already in `wp-config.php`):
```php
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);  // CRITICAL: false on production!
```

**Set up monitoring crons** (from Step 8, §8.7 and §8.8):
```bash
# CDN health check — every 15 minutes
(crontab -l 2>/dev/null; echo "*/15 * * * * /usr/local/bin/check-cdn-health.sh") | crontab -

# Unified error check — every hour
(crontab -l 2>/dev/null; echo "0 * * * * /usr/local/bin/check-all-errors.sh") | crontab -

# Verify crons are set
crontab -l | grep -E "cdn-health|check-all"
```

---

### 10.5 — Run Production Migration (Step 7)

> **This is the critical step.** Follow Step 7 exactly — it has the full procedure with cache warming, database URL rewriting, and verification.

```bash
# Enable maintenance mode
wp maintenance-mode activate

# Run cache warming (Step 7, §7.4)
wp eval-file /tmp/generate-urls.php > /tmp/all-media-urls.txt
bash cdn-cache-warm.sh

# Run database URL rewriting (Step 7, §7.5)
wp eval-file cdn-migrate.php

# Flush all caches (Step 7, §7.6)
wp cache flush
wp rocket clean --confirm 2>/dev/null || wp w3-total-cache flush all 2>/dev/null
sudo rm -rf /var/run/nginx-cache/* 2>/dev/null

# Disable maintenance mode
wp maintenance-mode deactivate
```

---

### 10.6 — Post-Deploy Verification (Immediate)

Run these checks **within 30 minutes** of deploy:

```bash
# ── 1. Site is up ──
curl -sI https://yourdomain.com | head -3
# Expected: HTTP/2 200

# ── 2. Zero old URLs remaining ──
wp eval 'global $wpdb;
echo "Posts: " . $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_content LIKE \"%yourdomain.com/wp-content/uploads%\"") . "\n";
echo "Meta:  " . $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->postmeta} WHERE meta_value LIKE \"%yourdomain.com/wp-content/uploads%\"") . "\n";'
# Expected: Posts: 0, Meta: 0

# ── 3. Origin serves files (no 301!) ──
curl -sI https://yourdomain.com/wp-content/uploads/2024/01/any-image.jpg | head -1
# Expected: HTTP/2 200 (NOT 301!)

# ── 4. CDN serves files ──
curl -sI https://cdn.yourdomain.com/wp-content/uploads/2024/01/any-image.jpg | grep -iE "^(HTTP|x-cache)"
# Expected: HTTP/2 200, x-cache: HIT

# ── 5. WebP working ──
curl -sI -H "Accept: image/webp" https://cdn.yourdomain.com/wp-content/uploads/2024/01/any-image.jpg | grep content-type
# Expected: content-type: image/webp

# ── 6. Maintenance mode off ──
curl -s https://yourdomain.com/ | grep -c "maintenance"
# Expected: 0

# ── 7. Quick visual check ──
echo "Open in browser and check images load:"
echo "  - Home page"
echo "  - A blog post with images"
echo "  - WP Admin → Media Library"
```

---

### 10.7 — Monitor for 48 Hours

**Hour 0–2 (Active monitoring — stay on SSH):**
```bash
# Watch logs in real-time
screen -S cdn-monitor
sudo tail -f /var/log/nginx/error.log &
tail -f /var/www/html/wp-content/debug.log &
# Detach: Ctrl+A then D
```

**Every 6 hours for 48 hours:**
```bash
# Quick health check
curl -sI https://cdn.yourdomain.com/wp-content/uploads/cdn-health-check.jpg | grep -E "HTTP|x-cache"
redis-cli ping
tail -1 /var/log/cdn-health.log
tail -5 /var/log/wp-optimization-errors.log
```

**After 24 hours — check:**
- BunnyCDN Dashboard → cache hit rate > 90%?
- Google Search Console → any new crawl errors?
- WP Admin → Site Kit → traffic normal?

---

### 10.8 — Final Performance Benchmark

After 48 hours (CDN cache fully warmed), run final benchmarks:

**Run PageSpeed Insights on 3 key pages:**
1. Home page
2. Most popular blog post
3. A page with many images

| Metric | Baseline (Step 1) | After Task 1 (Caching) | After Task 2 (CDN) | Target | Met? |
|---|---|---|---|---|---|
| TTFB (ms) | ___ | ___ | ___ | < 200ms | ☐ |
| Page Load (s) | ___ | ___ | ___ | < 3s | ☐ |
| Image Load (ms) | ___ | ___ | ___ | ≥ 50% faster | ☐ |
| PageSpeed Desktop | ___ /100 | ___ /100 | ___ /100 | ≥ 90 | ☐ |
| PageSpeed Mobile | ___ /100 | ___ /100 | ___ /100 | ≥ 80 | ☐ |
| LCP (s) | ___ | ___ | ___ | < 2.5s | ☐ |
| CDN Cache Hit Rate | N/A | N/A | ___ % | > 90% | ☐ |
| Object Cache Hit Rate | N/A | ___ % | ___ % | > 95% | ☐ |

---

## 🚨 Rollback Plan

If anything goes wrong after production deploy:

```bash
# ── LEVEL 1: CDN plugin issue (images broken) ──
# Deactivate CDN plugin → images revert to origin URLs
wp plugin deactivate cdn-enabler 2>/dev/null
# OR: WP Rocket → CDN tab → remove CDN URL → Save
# OR: W3TC → Performance → CDN → Disable → Save

# ── LEVEL 2: Migration script issue (wrong URLs in DB) ──
wp maintenance-mode activate
wp db import backup-prod-pre-cdn-YYYYMMDD-HHMM.sql
wp cache flush
wp maintenance-mode deactivate

# ── LEVEL 3: EWWW broke images ──
wp plugin deactivate ewww-image-optimizer
# Images revert to originals (EWWW keeps originals by default)

# ── LEVEL 4: Everything is broken ──
wp maintenance-mode activate
wp db import backup-prod-pre-cdn-YYYYMMDD-HHMM.sql
wp plugin deactivate cdn-enabler ewww-image-optimizer 2>/dev/null
wp cache flush
sudo nginx -t && sudo nginx -s reload
wp maintenance-mode deactivate
```

> **Rollback is fast** because with Pull Zone CDN, files never left your server. The only change was database URLs + plugin settings.

---

## ✅ Final Production Validation Checklist

| # | Check | How to Verify | ✓ |
|---|-------|---------------|---|
| 1 | Site is up and responding | `curl -sI https://yourdomain.com` → 200 | ☐ |
| 2 | All new uploads use CDN URL | Upload test image → src shows cdn.yourdomain.com | ☐ |
| 3 | All old images migrated | DB query → 0 rows with old origin URL | ☐ |
| 4 | Image load improvement ≥ 50% | PageSpeed/GTmetrix comparison | ☐ |
| 5 | LCP < 2.5 seconds | PageSpeed Insights | ☐ |
| 6 | HTTPS on all resources | DevTools Security tab → all secure | ☐ |
| 7 | WebP served to Chrome | Content-Type: image/webp | ☐ |
| 8 | JPEG fallback works | No WebP Accept → JPEG served | ☐ |
| 9 | Lazy loading active | `loading="lazy"` on images | ☐ |
| 10 | Responsive images (srcset) correct | Mobile viewport → smaller image loaded | ☐ |
| 11 | EWWW optimizing new uploads | Upload test image → shows "Optimized" | ☐ |
| 12 | EXIF metadata stripped | EWWW → Remove Metadata = ON | ☐ |
| 13 | Max image size 1920px | Upload 3000px image → stored ≤ 1920px | ☐ |
| 14 | CDN cache hit rate > 90% | BunnyCDN Dashboard | ☐ |
| 15 | Hotlinking protection active | External referer → 403 | ☐ |
| 16 | Original images protected | Direct access to original → 403 | ☐ |
| 17 | Origin serves files for CDN pull | `curl -I` origin → 200 (not 301) | ☐ |
| 18 | SSL valid on CDN subdomain | No warnings at https://cdn.yourdomain.com | ☐ |
| 19 | Media Library intact | WP Admin → Media → all files accessible | ☐ |
| 20 | No broken images | Crawl scan → 0 broken | ☐ |
| 21 | Site Kit connected to GA4 | WP Admin → Site Kit → data showing | ☐ |
| 22 | Query Monitor active | Admin toolbar shows page metrics | ☐ |
| 23 | Monitoring crons running | `crontab -l` shows both health scripts | ☐ |
| 24 | Log rotation configured | `/etc/logrotate.d/wordpress` exists | ☐ |
| 25 | No errors in logs after 48h | Error logs clean | ☐ |

---

## 📝 Deliverables — Task 2 Complete

Upon completing this step, deliver:

| Deliverable | Contents |
|---|---|
| **Migration Report** | # files migrated, # URL replacements, exceptions, migration log file |
| **Performance Report** | Before/after comparison table (§10.8), PageSpeed screenshots |
| **CDN Configuration Doc** | Pull Zone settings, SSL config, hotlinking rules, TTLs, DNS records |
| **Admin Guide** | How to: purge CDN cache, handle new uploads, check monitoring, run rollback |
| **Monitoring Dashboard** | BunnyCDN stats URL, UptimeRobot dashboard URL, Site Kit in WP Admin |

---

## 🏁 Project Sign-Off

| Task | Status |
|------|--------|
| Task 1 — WordPress Caching Configuration | ✅ COMPLETE |
| Task 2 — CDN Integration for Image Delivery | ✅ COMPLETE |
| Technical Requirements (4.1 Compatibility) | ✅ WP 5.8+, PHP 7.4+, Multisite |
| Technical Requirements (4.2 Security) | ✅ EXIF stripped, originals protected, HTTPS everywhere |
| Technical Requirements (4.3 Monitoring) | ✅ GA4, Query Monitor, error logging, CDN monitoring |
| Documentation Delivered | ✅ COMPLETE |
| Performance Targets Met | ✅ TTFB < 200ms, PageSpeed ≥ 90, Images ≥ 50% faster |

---

> ⬅️ [Previous: Step 9 — Staging QA & Performance Testing](step-09-staging-qa-performance.md) &nbsp;|&nbsp; 🏠 [Back to README](../README.md)

