# Task 2 — Step 7: Run Migration on Production

> **Estimated Duration:** 0.5 day (mostly waiting + monitoring)
> **Goal:** Execute the migration script (tested on staging in Step 6) on the production server, warm the CDN cache, and verify everything works.
> **Risk Level:** 🟠 Medium — database changes are involved. Rollback plan is essential.

---

## ⚠️ This Step Is a Production Run of Step 6

**Do NOT start here.** Step 6 contains the actual scripts and detailed explanations. This step is a **checklist-driven execution guide** for running those same scripts on production.

```
Step 6: Write scripts → test on staging → verify → ✅ all good
Step 7: Run the SAME scripts on production (this step)
```

---

## 🚦 Pre-Production Checklist

Do **NOT** proceed until every box is checked:

| # | Prerequisite | Verified? |
|---|---|---|
| 1 | Migration script tested on staging (Step 6) — 0 errors | ☐ |
| 2 | Staging images all load from CDN after migration | ☐ |
| 3 | CDN Pull Zone is set up and working (Step 2) | ☐ |
| 4 | Image optimization pipeline is active (Step 3) | ☐ |
| 5 | WordPress CDN plugin is configured (Step 4) | ☐ |
| 6 | CDN security configured — HTTPS, TLS 1.2+ (Step 5) | ☐ |
| 7 | Low-traffic maintenance window scheduled | ☐ |
| 8 | Team notified of maintenance window | ☐ |

---

## 🔧 Production Execution Steps

### 7.1 — Schedule Low-Traffic Window

```bash
# Find lowest traffic hour from Nginx logs
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f2 | sort | uniq -c | sort -rn | tail -5
# Pick the hour with lowest request count (usually 2:00-5:00 AM local time)
```

---

### 7.2 — Backup Production

```bash
ssh user@production-server
cd /var/www/html

# Verify you're on the RIGHT server
wp eval 'echo get_bloginfo("url") . "\n";'
# Must output: https://yourdomain.com (NOT staging!)

# Database backup
wp db export backup-production-pre-cdn-$(date +%Y%m%d-%H%M).sql

# Verify backup file
ls -lh backup-production-pre-cdn-*.sql
# Should be several MB at least

# Files backup (uploads directory)
tar -czf /tmp/backup-uploads-$(date +%Y%m%d-%H%M).tar.gz wp-content/uploads/

# Copy backups off-server
scp backup-production-pre-cdn-*.sql user@backup-server:/backups/
```

---

### 7.3 — Enable Maintenance Mode (Recommended)

```bash
# Prevents users from creating new content during migration
# (new posts during migration could have mixed URLs)
wp maintenance-mode activate
# Site shows: "Briefly unavailable for scheduled maintenance."
```

---

### 7.4 — Run Cache Warming Script

```bash
# Generate all image URLs (originals + all sizes)
# This is the same script from Step 6, §6.3
wp eval-file /tmp/generate-urls.php > /tmp/all-media-urls.txt 2>/tmp/url-gen-log.txt
cat /tmp/url-gen-log.txt

# Run the cache warming script (from Step 6, §6.3 Step B)
bash cdn-cache-warm.sh

# Verify cache is warm (spot check 5 random images)
ORIGIN="https://yourdomain.com"
CDN="https://cdn.yourdomain.com"
shuf -n 5 /tmp/all-media-urls.txt | sed "s|${ORIGIN}|${CDN}|g" | while read url; do
    echo "--- $url ---"
    curl -sI "$url" | grep -iE "^(x-cache|cdn-cache|age:)"
done
# Expected: x-cache: HIT for all 5
```

---

### 7.5 — Run Database URL Rewriting

```bash
# Dry run — check how many rows will be affected
wp eval 'global $wpdb;
$tables = [
    "posts (content)" => "SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_content LIKE \"%yourdomain.com/wp-content/uploads%\"",
    "posts (excerpt)" => "SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_excerpt LIKE \"%yourdomain.com/wp-content/uploads%\"",
    "postmeta"        => "SELECT COUNT(*) FROM {$wpdb->postmeta} WHERE meta_value LIKE \"%yourdomain.com/wp-content/uploads%\"",
    "options"         => "SELECT COUNT(*) FROM {$wpdb->options} WHERE option_value LIKE \"%yourdomain.com/wp-content/uploads%\" AND option_name NOT IN (\"siteurl\",\"home\",\"active_plugins\",\"template\",\"stylesheet\")",
];
foreach ($tables as $name => $sql) {
    echo $name . ": " . $wpdb->get_var($sql) . " rows\n";
}'

# Review the counts — do they match staging?
# If yes, run the migration:
wp eval-file cdn-migrate.php

# Expected output:
# Step 1: Updating wp_posts.post_content...
#   → Updated 847 posts
# Step 2: Updating wp_posts.post_excerpt...
#   → Updated 12 post excerpts
# Step 3: Updating wp_postmeta...
#   → Updated 1203 postmeta rows
# Step 4: Updating wp_options...
#   → Updated 15 option rows
# Step 5: WordPress object cache flushed
# ✅ Migration complete!
```

---

### 7.6 — Flush All Caches

```bash
# Object cache (Redis)
wp cache flush

# Page cache (WP Rocket)
wp rocket clean --confirm 2>/dev/null
# OR W3 Total Cache
wp w3-total-cache flush all 2>/dev/null

# FastCGI cache (if using Nginx FastCGI)
# Only if you have the Nginx Helper plugin:
# It auto-purges on content change, but force it:
sudo rm -rf /var/run/nginx-cache/*

# Verify Nginx config is still valid after any changes
sudo nginx -t && sudo nginx -s reload
```

---

### 7.7 — Disable Maintenance Mode

```bash
wp maintenance-mode deactivate
```

---

### 7.8 — Post-Migration Verification

Run these checks **immediately** after migration:

```bash
# ── 1. Zero old URLs remaining ──
wp eval 'global $wpdb;
echo "Posts: " . $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_content LIKE \"%yourdomain.com/wp-content/uploads%\"") . "\n";
echo "Meta:  " . $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->postmeta} WHERE meta_value LIKE \"%yourdomain.com/wp-content/uploads%\"") . "\n";'
# Expected: Posts: 0, Meta: 0

# ── 2. Origin still serves files (no 301 redirect!) ──
curl -sI https://yourdomain.com/wp-content/uploads/2024/01/any-known-image.jpg | head -5
# Expected: HTTP/2 200 (NOT 301 or 302!)

# ── 3. CDN serves files ──
curl -sI https://cdn.yourdomain.com/wp-content/uploads/2024/01/any-known-image.jpg | grep -iE "^(HTTP|x-cache|content-type)"
# Expected: HTTP/2 200, x-cache: HIT, content-type: image/jpeg

# ── 4. WebP served through CDN (if EWWW configured) ──
curl -sI -H "Accept: image/webp" https://cdn.yourdomain.com/wp-content/uploads/2024/01/any-known-image.jpg | grep content-type
# Expected: content-type: image/webp

# ── 5. Spot-check 5 pages in browser ──
echo "Open these pages and check images load:"
wp post list --post_type=post --post_status=publish --fields=ID,post_title,url --format=table | head -6
# Right-click any image → Inspect → src should be https://cdn.yourdomain.com/...
```

---

### 7.9 — Monitor for 48 Hours

```bash
# Watch for errors in real-time (run in screen/tmux)
screen -S cdn-monitor

# Nginx errors
sudo tail -f /var/log/nginx/error.log &

# PHP/WordPress errors
tail -f /var/www/html/wp-content/debug.log &

# Check for 404s every hour
watch -n 3600 'grep "404" /var/log/nginx/access.log | grep "cdn\|uploads" | tail -10'

# Detach: Ctrl+A then D
# Reconnect: screen -r cdn-monitor
```

**Also check:**
- BunnyCDN Dashboard → Statistics → Error Responses → filter by 404
- Google Search Console → Coverage → check for new crawl errors after 24h
- PageSpeed Insights → re-run on 3 key pages

---

## 🚨 Rollback Plan

If anything goes wrong, restore within minutes:

```bash
# ── Step 1: Restore database (reverts ALL URL changes) ──
wp maintenance-mode activate
wp db import backup-production-pre-cdn-YYYYMMDD-HHMM.sql

# ── Step 2: Flush all caches ──
wp cache flush
wp rocket clean --confirm 2>/dev/null

# ── Step 3: Deactivate CDN plugin (stops URL rewriting) ──
wp plugin deactivate cdn-enabler 2>/dev/null      # if using CDN Enabler
# OR: WP Rocket → CDN tab → remove CDN URL
# OR: W3TC → CDN → disable

# ── Step 4: Verify site works without CDN ──
wp maintenance-mode deactivate
curl -sI https://yourdomain.com | head -3
# Should load normally with images from origin

# ── Step 5: Investigate what went wrong ──
# Check migration log: /var/www/html/wp-content/cdn-migration-log-*.txt
# Check CDN warm log: /tmp/cdn-warm-*.log
# Check Nginx error log: /var/log/nginx/error.log
```

> **Rollback is fast because:** With Pull Zone CDN, files never left your server. The only change was database URLs. Restoring the database backup = instant rollback.

---

## ✅ Validation Checklist

| # | Check | How to Verify | ✓ |
|---|-------|---------------|---|
| 1 | Production database backed up | Backup SQL file exists + copied off-server | ☐ |
| 2 | Uploads directory backed up | tar.gz file exists | ☐ |
| 3 | Cache warming completed | Warm log shows all files, failures < 1% | ☐ |
| 4 | CDN returns HIT for warmed files | `curl -sI` on 5 random CDN URLs | ☐ |
| 5 | Migration script ran without errors | Terminal output shows ✅ complete | ☐ |
| 6 | 0 posts with old origin URLs | `wp eval` query returns 0 | ☐ |
| 7 | 0 postmeta with old origin URLs | `wp eval` query returns 0 | ☐ |
| 8 | Origin serves files (no 301!) | `curl -I` origin URL → 200 OK | ☐ |
| 9 | CDN serves files | `curl -I` CDN URL → 200 OK | ☐ |
| 10 | Images load on 5+ production pages | Visual spot-check in browser | ☐ |
| 11 | Media Library thumbnails visible | WP Admin → Media → all thumbnails load | ☐ |
| 12 | All caches flushed | Object + page + FastCGI caches cleared | ☐ |
| 13 | Maintenance mode disabled | Site publicly accessible | ☐ |
| 14 | No new errors in logs after 6h | Nginx + PHP error logs clean | ☐ |
| 15 | Migration log saved | `/wp-content/cdn-migration-log-*.txt` exists | ☐ |

---

> ⬅️ [Previous: Step 6 — Migration Script](step-06-media-migration-script.md) &nbsp;|&nbsp; ➡️ [Next: Step 8 — Analytics & Monitoring](step-08-analytics-monitoring.md)

