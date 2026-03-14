# Task 2 — Step 8: Analytics, Monitoring & Error Logging

> **Estimated Duration:** 0.5 day
> **Goal:** Set up monitoring so you can see CDN health, page speed improvements, cache performance, and errors — all from WordPress Admin + one external dashboard.
> **Risk Level:** ✅ Low — monitoring is read-only, nothing can break.

---

## 📋 Project Requirement (Section 4.3)

> **4.3. Monitoring and analytics**
> - Integration with Google Analytics
> - Performance monitoring configuration
> - Caching and CDN error logging

### Where Each Requirement Is Covered

| Requirement | Solution | Dashboard |
|---|---|---|
| **Google Analytics integration** | Site Kit by Google plugin (§8.1) | ✅ WP Admin |
| **Performance monitoring** | Query Monitor plugin (§8.2) + PageSpeed (§8.4) | ✅ WP Admin |
| **Caching error logging** | WP Debugging plugin + debug.log (§8.3) | ✅ WP Admin |
| **CDN error logging** | BunnyCDN built-in dashboard (§8.5) | External web panel |
| **Alerting** | Cron scripts + UptimeRobot (§8.6, §8.7) | Email alerts |

---

## 🧠 The Monitoring Stack (All Free)

```
┌──────────────────────────────────────────────────────────────┐
│  WordPress Admin Dashboard (check daily)                      │
│                                                               │
│  ┌─────────────┐  ┌────────────────┐  ┌───────────────────┐  │
│  │  Site Kit    │  │ Query Monitor  │  │  WP Debugging     │  │
│  │             │  │                │  │                   │  │
│  │ • GA4 stats │  │ • Page load ms │  │ • PHP errors      │  │
│  │ • Web Vitals│  │ • DB queries   │  │ • Plugin warnings │  │
│  │ • Traffic   │  │ • Cache hits   │  │ • Fatal errors    │  │
│  │ • PageSpeed │  │ • Redis status │  │ • Deprecated calls│  │
│  └─────────────┘  └────────────────┘  └───────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│  External (check weekly)                                      │
│                                                               │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │  BunnyCDN Panel   │  │  UptimeRobot     │                  │
│  │                   │  │                  │                  │
│  │ • Cache hit rate  │  │ • CDN uptime %   │                  │
│  │ • Bandwidth       │  │ • Response time  │                  │
│  │ • 404/5xx errors  │  │ • Email alerts   │                  │
│  │ • Origin requests │  │                  │                  │
│  └──────────────────┘  └──────────────────┘                  │
├──────────────────────────────────────────────────────────────┤
│  Automated Alerts (runs 24/7)                                 │
│                                                               │
│  • CDN health check cron     → emails if CDN returns non-200 │
│  • Unified error check cron  → emails if PHP/Nginx errors    │
│  • UptimeRobot               → emails if CDN endpoint is down│
└──────────────────────────────────────────────────────────────┘
```

---

## 🔧 Part 1: WordPress Admin Plugins

### 8.1 — Site Kit by Google (GA4 + PageSpeed in WP Admin)

**What it does:** Shows Google Analytics, PageSpeed Insights, and Search Console data directly inside your WordPress admin — no need to open separate Google dashboards.

```bash
# Install Site Kit (official Google plugin, free)
wp plugin install google-site-kit --activate

# For Multisite — network activate:
wp plugin install google-site-kit --activate-network
```

**Setup (one-time, ~5 minutes):**

1. Go to **WP Admin → Site Kit → Start Setup**
2. Sign in with your Google account
3. Grant permissions (read-only access to your Google services)
4. Connect these services:
   - ✅ **Google Analytics (GA4)** — traffic, user behavior, events
   - ✅ **Search Console** — search rankings, crawl errors
   - ✅ **PageSpeed Insights** — Core Web Vitals, performance score

**What you see in WP Admin after setup:**

```
Site Kit Dashboard (WP Admin → Site Kit):
┌────────────────────────────────────────────────┐
│  All Traffic          │  PageSpeed              │
│  • 12,450 sessions    │  • Mobile: 82/100       │
│  • 8,200 users        │  • Desktop: 95/100      │
│  • 2.1% bounce rate   │  • LCP: 1.8s ✅         │
│                       │  • CLS: 0.05 ✅         │
├───────────────────────┼─────────────────────────┤
│  Search Console       │  Speed Insights         │
│  • 45,000 impressions │  • "Serve images in     │
│  • 1,200 clicks       │    next-gen formats" ✅  │
│  • 3.2% CTR           │  • "Properly size        │
│                       │    images" ✅             │
└───────────────────────┴─────────────────────────┘
```

**Track CDN impact — compare before/after:**

```
GA4 → Reports → Engagement → Pages and Screens

Compare date ranges:
  Before CDN: [date you started] to [CDN launch date]
  After CDN:  [CDN launch date] to [today]

Key metrics to compare:
  • Average Page Load Time  (should decrease)
  • Bounce Rate            (should decrease)
  • LCP                    (should decrease — biggest win from CDN)
  • Pages per Session      (should increase — faster = more browsing)
```

> **⚠️ Note:** Site Kit shows data from Google's services. There's a 24-48 hour delay for new data to appear. Don't expect real-time stats on launch day.

---

### 8.2 — Query Monitor (Real-Time Performance Debugging)

**What it does:** Adds a toolbar panel to every page showing exactly what happened server-side: database queries, cache hits/misses, PHP errors, HTTP API calls, and more. Essential for debugging performance issues.

```bash
# Install Query Monitor (free, by John Blackbourn — WordPress core contributor)
wp plugin install query-monitor --activate
```

**What you see on every page (admin toolbar):**

```
┌─────────────────────────────────────────────────────────────┐
│ Admin Bar → "3.2s | 142 queries | 24.5MB"  ← Click to expand │
├─────────────────────────────────────────────────────────────┤
│ Overview     │ Page generation time: 3.2s                    │
│              │ Peak memory: 24.5MB                           │
│              │ Database queries: 142 (0.8s total)            │
├──────────────┼──────────────────────────────────────────────┤
│ Queries      │ Lists every SQL query with:                   │
│              │ • Time taken (highlight slow ones > 0.05s)    │
│              │ • Caller (which plugin/theme ran it)           │
│              │ • Duplicate detection (same query run twice)   │
├──────────────┼──────────────────────────────────────────────┤
│ Object Cache │ Cache hits: 847  ← Redis is working!          │
│              │ Cache misses: 23                               │
│              │ Cache hit rate: 97.3%                          │
│              │ ⚠️ If you see "No object cache" → Redis       │
│              │    drop-in isn't installed (check Step 3 T1)   │
├──────────────┼──────────────────────────────────────────────┤
│ PHP Errors   │ Shows any warnings/notices on current page    │
│              │ Click to see file + line number                │
├──────────────┼──────────────────────────────────────────────┤
│ HTTP API     │ Shows external HTTP calls made during page     │
│              │ load (e.g., plugin update checks, API calls)  │
│              │ ⚠️ Too many = slow page load                  │
└──────────────┴──────────────────────────────────────────────┘
```

**Key things to check with Query Monitor:**

| What to Check | Where in QM | What's Healthy | Red Flag |
|---|---|---|---|
| Object cache hits | Object Cache tab | Hit rate > 95% | "No object cache" or hit rate < 80% |
| Slow queries | Queries → sort by time | All < 0.05s | Any query > 0.5s |
| Duplicate queries | Queries → Dupes tab | 0 duplicates | 10+ duplicates = bad plugin |
| HTTP API calls | HTTP API Calls tab | < 5 per page | 20+ calls = plugin bloat |
| PHP errors | PHP Errors tab | 0 errors | Any Fatal or Warning |

> **⚠️ Security:** Query Monitor only shows data to **administrators**. Regular visitors never see it. But still — **disable it on production if not actively debugging**, as it adds minor overhead to every page load.

```bash
# To disable when not debugging:
wp plugin deactivate query-monitor

# Re-enable when needed:
wp plugin activate query-monitor
```

---

### 8.3 — WP Debugging (PHP Error Log Viewer in WP Admin)

**The problem:** WordPress logs PHP errors to `/wp-content/debug.log`, but you need SSH access to read it. WP Debugging lets you view it from WP Admin.

```bash
# Install WP Debugging (free)
wp plugin install wp-debugging --activate
```

**Enable debug logging in `wp-config.php`:**

```php
// Add BEFORE the line: "That's all, stop editing!"

// Enable error logging (REQUIRED for WP Debugging plugin to work)
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);       // Logs to /wp-content/debug.log
define('WP_DEBUG_DISPLAY', false);  // CRITICAL: Do NOT show errors to visitors!
define('SCRIPT_DEBUG', false);      // Keep minified scripts in production
```

> **⚠️ `WP_DEBUG_DISPLAY` must be `false` on production!** If set to `true`, PHP errors (including file paths and database info) are shown to every visitor — security risk.

**What you see in WP Admin:**

```
WP Admin → Tools → WP Debugging

┌──────────────────────────────────────────────────────────┐
│ debug.log viewer                                          │
│                                                           │
│ [2026-03-14 03:15:22] PHP Warning: Undefined variable    │
│   $image_url in /themes/mytheme/header.php on line 42    │
│                                                           │
│ [2026-03-14 03:15:23] PHP Notice: wp_get_attachment_url  │
│   was called incorrectly in /plugins/cdn-enabler/cdn.php │
│                                                           │
│ [2026-03-14 04:00:01] PHP Fatal error: Allowed memory    │
│   size of 268435456 bytes exhausted                       │
│   in /plugins/ewww-image-optimizer/bulk.php on line 891  │
│                                                           │
│ [Clear Log]  [Download Log]  [Auto-refresh: ON]          │
└──────────────────────────────────────────────────────────┘
```

**What to watch for after CDN migration:**

| Error Pattern | What It Means | Action |
|---|---|---|
| `Failed opening 'https://cdn...'` | PHP trying to fetch CDN URL as local file | Fix: file path vs URL confusion in theme/plugin |
| `Undefined variable $image_url` | CDN plugin incompatibility with theme | Fix: update theme image functions |
| `Allowed memory size exhausted` | EWWW bulk optimization running out of memory | Fix: increase `memory_limit` or use WP-CLI |
| `cURL error 28: Connection timed out` | CDN or origin timeout | Check: CDN status + origin server health |

---

### 8.4 — Simple History (Optional — WordPress Action Log)

**What it does:** Logs every significant action in WordPress: who installed a plugin, who edited a post, login attempts, setting changes. Useful for auditing "who changed what" after CDN setup.

```bash
# Install Simple History (free)
wp plugin install simple-history --activate
```

**Shows in WP Admin → Dashboard → Simple History widget:**
```
• admin activated plugin "cdn-enabler" — 2 hours ago
• admin updated option "cdn_enabler_settings" — 2 hours ago  
• admin ran "wp eval-file cdn-migrate.php" — 3 hours ago
• editor updated post "About Us" — 5 hours ago
```

> This is optional but helpful when debugging: "the site broke after someone changed a CDN setting at 3pm."

---

## 🔧 Part 2: External Monitoring

### 8.5 — BunnyCDN Dashboard (CDN Metrics)

This is already included with your BunnyCDN account — no plugin needed.

**BunnyCDN → Your Pull Zone → Statistics:**

```
Key metrics to monitor:

Cache Hit Rate:     Target > 90%
                    If < 80% → check Cache-Control headers, TTL settings
                    If < 50% → something is seriously wrong (check Vary headers)

Bandwidth:          Shows how much data CDN served
                    Compare with origin bandwidth to see CDN savings

Error Rate:         Target < 1%
                    4xx errors = missing files (check migration script)
                    5xx errors = origin server issues

Origin Shield:      Shows requests that went back to your server
                    Should decrease over time as cache warms
```

**Enable CDN logging:**

```
BunnyCDN → Your Pull Zone → Security → Logging:
  ✅ Enable Logging
  Log Format:  Extended (includes cache status, response time)
```

**Alert thresholds:**

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Cache Hit Rate | > 90% | 80-90% | < 80% |
| 5xx Error Rate | < 0.1% | 0.1-1% | > 1% |
| Origin Request Rate | < 10% | 10-30% | > 30% |
| Avg Response Time | < 100ms | 100-500ms | > 500ms |

> **How often to check:** Daily for the first week after CDN launch, then weekly.

---

### 8.6 — UptimeRobot (CDN Uptime Monitoring — Free)

Monitors your CDN endpoint every 5 minutes and emails you if it goes down.

**Setup (5 minutes):**

1. Go to [uptimerobot.com](https://uptimerobot.com) → create free account
2. Add monitor:
   ```
   Monitor Type:    HTTPS
   Friendly Name:   CDN - yourdomain.com
   URL:             https://cdn.yourdomain.com/wp-content/uploads/cdn-health-check.jpg
   Monitoring Interval: 5 minutes
   Alert Contacts:  your@email.com
   ```
3. Create a sentinel image for the health check:
   ```bash
   # Create a tiny 1x1 pixel image at a known URL
   echo -e '\x89PNG\r\n\x1a\n' > /var/www/html/wp-content/uploads/cdn-health-check.jpg
   # Or just use any existing small image URL
   ```

**Free tier includes:** 50 monitors, 5-minute intervals, email alerts.

---

## 🔧 Part 3: Automated Server-Side Alerts

### 8.7 — CDN Health Check Cron

```bash
cat > /usr/local/bin/check-cdn-health.sh << 'EOF'
#!/bin/bash
# Checks CDN endpoint every 15 minutes, emails on failure
CDN_URL="https://cdn.yourdomain.com/wp-content/uploads/cdn-health-check.jpg"
LOG="/var/log/cdn-health.log"
ALERT_EMAIL="your@email.com"

STATUS=$(curl -o /dev/null -s -w "%{http_code}" --max-time 10 "$CDN_URL")
RESPONSE_TIME=$(curl -o /dev/null -s -w "%{time_total}" --max-time 10 "$CDN_URL")

if [ "$STATUS" != "200" ]; then
    echo "FAIL: HTTP $STATUS at $(date) (${RESPONSE_TIME}s)" >> "$LOG"
    echo "CDN health check FAILED: HTTP $STATUS at $(date)" | \
        mail -s "🔴 CDN DOWN: yourdomain.com" "$ALERT_EMAIL" 2>/dev/null
else
    echo "OK: HTTP $STATUS at $(date) (${RESPONSE_TIME}s)" >> "$LOG"
fi
EOF

chmod +x /usr/local/bin/check-cdn-health.sh

# Run every 15 minutes
(crontab -l 2>/dev/null; echo "*/15 * * * * /usr/local/bin/check-cdn-health.sh") | crontab -
```

---

### 8.8 — Unified Error Monitoring Cron

Checks all error sources (WordPress, Nginx, Redis) hourly and emails if anything is wrong:

```bash
cat > /usr/local/bin/check-all-errors.sh << 'ENDSCRIPT'
#!/bin/bash
# Unified error check — runs every hour via cron
LOG="/var/log/wp-optimization-errors.log"
ALERT_EMAIL="your@email.com"
ERRORS_FOUND=0

echo "=== Error Check $(date) ===" >> "$LOG"

# 1. WordPress PHP errors (last hour)
WP_LOG="/var/www/html/wp-content/debug.log"
if [ -f "$WP_LOG" ]; then
    WP_ERRORS=$(find "$WP_LOG" -mmin -60 -exec grep -c "PHP Fatal\|PHP Warning\|PHP Error" {} \; 2>/dev/null || echo 0)
    if [ "$WP_ERRORS" -gt 0 ]; then
        echo "  WordPress: $WP_ERRORS PHP errors in last hour" >> "$LOG"
        ERRORS_FOUND=1
    fi
fi

# 2. Nginx 5xx errors (last hour)
NGINX_5XX=$(awk -v d="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '$4 ~ d && $9 ~ /^5/' /var/log/nginx/access.log 2>/dev/null | wc -l)
if [ "$NGINX_5XX" -gt 0 ]; then
    echo "  Nginx 5xx: $NGINX_5XX errors in last hour" >> "$LOG"
    ERRORS_FOUND=1
fi

# 3. Redis status
REDIS_OK=$(redis-cli ping 2>/dev/null)
if [ "$REDIS_OK" != "PONG" ]; then
    echo "  Redis: NOT RESPONDING" >> "$LOG"
    ERRORS_FOUND=1
else
    REDIS_MEM=$(redis-cli info memory 2>/dev/null | grep used_memory_human | cut -d: -f2 | tr -d '[:space:]')
    echo "  Redis: OK (memory: $REDIS_MEM)" >> "$LOG"
fi

# 4. FastCGI cache hit rate (last 1000 requests)
if [ -f /var/log/nginx/access.log ]; then
    TOTAL_REQ=$(tail -1000 /var/log/nginx/access.log | wc -l)
    CACHE_HITS=$(tail -1000 /var/log/nginx/access.log | grep -c "HIT" 2>/dev/null || echo 0)
    if [ "$TOTAL_REQ" -gt 0 ]; then
        HIT_RATE=$((CACHE_HITS * 100 / TOTAL_REQ))
        echo "  FastCGI cache hit rate: ${HIT_RATE}% (${CACHE_HITS}/${TOTAL_REQ})" >> "$LOG"
        if [ "$HIT_RATE" -lt 50 ]; then
            echo "  ⚠️ Cache hit rate below 50%!" >> "$LOG"
            ERRORS_FOUND=1
        fi
    fi
fi

# 5. Disk space check (uploads directory)
DISK_USAGE=$(df /var/www/html/wp-content/uploads 2>/dev/null | tail -1 | awk '{print $5}' | tr -d '%')
if [ -n "$DISK_USAGE" ] && [ "$DISK_USAGE" -gt 85 ]; then
    echo "  Disk: ${DISK_USAGE}% used — running low!" >> "$LOG"
    ERRORS_FOUND=1
fi

# Send alert if any errors found
if [ "$ERRORS_FOUND" -eq 1 ]; then
    tail -20 "$LOG" | mail -s "⚠️ WP Optimization Alert: $(hostname)" "$ALERT_EMAIL" 2>/dev/null
fi
ENDSCRIPT

chmod +x /usr/local/bin/check-all-errors.sh

# Run every hour
(crontab -l 2>/dev/null; echo "0 * * * * /usr/local/bin/check-all-errors.sh") | crontab -
```

---

### 8.9 — Log Rotation (Prevent Disk Full)

All these logs will grow forever without rotation:

```bash
# WordPress debug.log rotation
cat > /etc/logrotate.d/wordpress << 'EOF'
/var/www/html/wp-content/debug.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    create 0644 www-data www-data
}
EOF

# CDN health log rotation
cat >> /etc/logrotate.d/wordpress << 'EOF'

/var/log/cdn-health.log
/var/log/wp-optimization-errors.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    create 0644 root root
}
EOF
```

---

## 🔧 Part 4: Error Log Locations — Quick Reference

When debugging issues, check these locations:

| Component | Log File | What It Catches | View From WP Admin? |
|---|---|---|---|
| **WordPress PHP** | `/wp-content/debug.log` | PHP errors, plugin conflicts, deprecated functions | ✅ WP Debugging plugin |
| **PHP-FPM** | `/var/log/php8.1-fpm.log` | Fatal errors, memory limits, timeouts | ❌ SSH only |
| **Nginx errors** | `/var/log/nginx/error.log` | 502s, socket failures, config errors | ❌ SSH only |
| **Nginx access** | `/var/log/nginx/access.log` | 404s, cache HIT/MISS stats | ❌ SSH only |
| **Redis** | `/var/log/redis/redis-server.log` | Connection failures, memory warnings | ❌ SSH only |
| **CDN (BunnyCDN)** | BunnyCDN Dashboard → Logging | 4xx/5xx from CDN edge, origin timeouts | ❌ BunnyCDN web panel |
| **WordPress actions** | WP Admin → Simple History | Plugin changes, setting updates, logins | ✅ Simple History plugin |
| **CDN health** | `/var/log/cdn-health.log` | CDN endpoint status checks | ❌ SSH only |
| **Unified errors** | `/var/log/wp-optimization-errors.log` | All error sources combined | ❌ SSH only (but emailed) |

---

## 📋 Daily vs Weekly Monitoring Guide

### Daily (First 2 Weeks After CDN Launch)

```bash
# 1. Quick check — run from your laptop/SSH
ssh user@production-server

# Redis alive?
redis-cli ping          # Expected: PONG

# Any PHP errors today?
grep "$(date +%Y-%m-%d)" /var/www/html/wp-content/debug.log | grep -c "Fatal\|Error"
# Expected: 0

# Any 5xx errors today?
grep "$(date +%d/%b/%Y)" /var/log/nginx/access.log | awk '$9 ~ /^5/' | wc -l
# Expected: 0

# CDN health log
tail -5 /var/log/cdn-health.log
# Expected: all "OK"
```

```
# 2. WP Admin checks (2 minutes)
- Site Kit dashboard    → traffic normal? PageSpeed score stable?
- Query Monitor         → object cache hit rate > 95%?
- WP Debugging          → any new PHP errors?
- Media Library         → images loading from CDN?
```

### Weekly (After First 2 Weeks)

```
# External dashboards
- BunnyCDN Statistics   → cache hit rate > 90%? error rate < 1%?
- UptimeRobot           → 100% uptime?
- Google Search Console → any new crawl errors?

# Performance comparison
- PageSpeed Insights    → run on 3 key pages → scores stable/improving?
```

---

## 📊 Performance Baseline Comparison

Record these metrics now (after CDN) and compare with the baseline from Task 1 Step 1:

| Metric | Before CDN | After CDN | Target | Improvement |
|---|---|---|---|---|
| Page Load Time | ___ s | ___ s | < 3s | ___ % |
| LCP (Largest Contentful Paint) | ___ s | ___ s | < 2.5s | ___ % |
| CLS (Cumulative Layout Shift) | ___ | ___ | < 0.1 | ___ |
| PageSpeed Score (Mobile) | ___ /100 | ___ /100 | > 80 | ___ pts |
| PageSpeed Score (Desktop) | ___ /100 | ___ /100 | > 90 | ___ pts |
| CDN Cache Hit Rate | N/A | ___ % | > 90% | — |
| Origin Bandwidth | ___ GB/mo | ___ GB/mo | -70% | ___ % |
| Object Cache Hit Rate | N/A | ___ % | > 95% | — |

> **Fill in the "Before" column from the audit results in Task 1 Step 1 and Task 2 Step 1.**

---

## ✅ Validation Checklist

| # | Check | How to Verify | ✓ |
|---|-------|---------------|---|
| 1 | Site Kit installed and connected to GA4 | WP Admin → Site Kit → shows traffic data | ☐ |
| 2 | Query Monitor installed | Admin toolbar shows page load time + query count | ☐ |
| 3 | Object cache hit rate > 95% | Query Monitor → Object Cache tab | ☐ |
| 4 | WP Debugging installed | WP Admin → Tools → WP Debugging shows log | ☐ |
| 5 | `WP_DEBUG_LOG` enabled | `grep WP_DEBUG wp-config.php` shows `true` | ☐ |
| 6 | `WP_DEBUG_DISPLAY` is `false` | Visitors do NOT see PHP errors | ☐ |
| 7 | BunnyCDN logging enabled | BunnyCDN → Logging tab → logs visible | ☐ |
| 8 | CDN cache hit rate > 90% | BunnyCDN → Statistics | ☐ |
| 9 | UptimeRobot monitoring CDN endpoint | UptimeRobot dashboard shows "Up" | ☐ |
| 10 | CDN health check cron running | `crontab -l` shows `check-cdn-health.sh` | ☐ |
| 11 | Unified error cron running | `crontab -l` shows `check-all-errors.sh` | ☐ |
| 12 | Log rotation configured | `cat /etc/logrotate.d/wordpress` exists | ☐ |
| 13 | PageSpeed score recorded | Before/After comparison table filled in | ☐ |
| 14 | Simple History installed (optional) | WP Admin → Dashboard → activity log visible | ☐ |

---

> ⬅️ [Previous: Step 7 — Migrate Images to CDN](step-07-migrate-images-to-cdn.md) &nbsp;|&nbsp; ➡️ [Next: Step 9 — Staging QA & Performance Testing](step-09-staging-qa-performance.md)

