# 🚀 WordPress Performance Optimization — Step-by-Step Guides

> **Project:** Optimizing the performance of the WordPress website
> **Total Budget:** 30 Business Days
> **Generated:** March 13, 2026

---

## 📌 Technical Requirements

> 📊 **New here? Start with the [Plugin vs Manual Overview](OVERVIEW-plugin-vs-manual.md)** — see what the caching plugin handles automatically vs what you must configure manually on the server.

> All steps in this guide must satisfy these requirements. If a step conflicts with any requirement below, the requirement wins.

### 4.1 — Compatibility

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| WordPress | **5.8+** | Native lazy loading (`loading="lazy"`) requires WP 5.5+. WebP support in Media Library requires WP 5.8+. Block editor image handling requires WP 5.8+. |
| PHP | **7.4+** | PHP 7.4 is the minimum for Redis Object Cache plugin (v2.1+), Imagify, and modern WP Rocket versions. PHP 8.0+ recommended for performance (~20% faster than 7.4). PHP 8.1+ recommended for production (7.4 is EOL since Nov 2022). |
| Multisite | **Supported** | All caching and CDN configs must work on both single-site and Multisite. Multisite requires unique Redis key prefixes per sub-site, per-site cache purge rules, and CDN URL rewriting that respects sub-site paths (`/site1/wp-content/uploads/` vs `/site2/wp-content/uploads/`). |
| Plugin/theme compatibility | **Required** | All caching must be tested with existing active plugins and themes. Page builders (Elementor, Divi), WooCommerce, membership plugins, and contact form plugins are the most common conflict sources. |

### 4.2 — Security

| Requirement | Description |
|-------------|-------------|
| Image privacy | Private/draft post images and membership-gated content must NOT be publicly accessible on the CDN. Uploads directory must have proper access controls. |
| Direct access protection | Original full-resolution images should be protected from direct hotlinking and unauthorized scraping. CDN must serve optimized versions; originals should be access-controlled on origin. |
| Secure data transfer | All communication between origin server and CDN must use HTTPS (TLS 1.2+). CDN pull requests to origin must go over HTTPS. No unencrypted image transfer at any point. |

### 4.3 — Monitoring & Analytics

| Requirement | Description |
|-------------|-------------|
| Google Analytics integration | GA4 Core Web Vitals (LCP, FID/INP, CLS) must be tracked before and after optimization. CDN image delivery metrics should be tagged for comparison. |
| Performance monitoring | Continuous monitoring of TTFB, cache hit rate, CDN response time, and origin offload percentage. Alerts for degradation. |
| Error logging | Centralized logging of: WordPress errors (`debug.log`), PHP-FPM errors, Nginx errors, Redis connection failures, and CDN 4xx/5xx errors. Alerts for error rate spikes. |

---

## 📁 Directory Structure

```
web-optimization/
│
├── task-1-caching/
│   ├── step-01-audit-current-setup.md
│   ├── step-02-page-caching-plugin.md
│   ├── step-03-object-caching-redis.md
│   ├── step-04-browser-caching.md
│   ├── step-05-minification-concatenation.md
│   ├── step-06-server-level-caching.md
│   ├── step-07-staging-testing-qa.md
│   └── step-08-deploy-production.md
│
└── task-2-cdn/
    ├── step-01-audit-media-library.md
    ├── step-02-cdn-provider-setup.md
    ├── step-03-image-optimization-pipeline.md
    ├── step-04-wordpress-cdn-plugin.md
    ├── step-05-cdn-caching-security.md
    ├── step-06-media-migration-script.md
    ├── step-07-migrate-images-to-cdn.md
    ├── step-08-analytics-monitoring.md
    ├── step-09-staging-qa-performance.md
    └── step-10-deploy-production.md
```

---

## 📋 Task Summary

| Task | Steps | Estimated Time | Complexity |
|------|-------|---------------|------------|
| **Task 1:** WordPress Caching Configuration | 8 steps | 8–10 Business Days | 🟡 Medium |
| **Task 2:** CDN Integration for Image Delivery | 10 steps | 12–14 Business Days | 🔴 High |

---

## ⏱️ Estimation Breakdown by Step

### Task 1 — WordPress Caching Configuration (8–10 Business Days)

| Step | Title | Estimated Time | Risk | What You'll Do |
|---|---|---|---|---|
| **1** | Audit Current Setup | 0.5 day | ✅ Safe | Run diagnostics, record baseline PageSpeed, check PHP/WP versions |
| **2** | Page Caching Plugin | 1 day | 🟡 Medium | Choose WP Rocket or W3TC, install, configure exclusions (WooCommerce!) |
| **3** | Object Caching (Redis) | 1.5 days | 🟠 High | Install Redis server, configure wp-config.php, verify graceful degradation |
| **4** | Browser Caching | 1 day | 🟠 High | Nginx or Apache config for static asset cache headers, Gzip/Brotli |
| **5** | Minification & Concatenation | 0.5 day | 🟡 Medium | Enable in caching plugin, test each feature one-by-one for JS breakage |
| **6** | Server-Level Caching (FastCGI) | 2 days | 🔴 Critical | Nginx FastCGI cache config — most impactful but most dangerous step |
| **7** | Staging Testing & QA | 1.5 days | ✅ Safe | Test everything on staging, run benchmarks, verify cache behavior |
| **8** | Deploy to Production | 1 day | 🔴 Critical | Production deploy with rollback plan, 24h monitoring |
| | **Task 1 Total** | **9 days** | | |

### Task 2 — CDN Integration for Image Delivery (12–14 Business Days)

| Step | Title | Estimated Time | Risk | What You'll Do |
|---|---|---|---|---|
| **1** | Audit Media Library | 0.5 day | ✅ Safe | Count images, measure sizes, identify large files, check upload structure |
| **2** | CDN Provider Setup | 1 day | 🟠 High | Create BunnyCDN Pull Zone, configure DNS (CNAME), SSL, verify propagation |
| **3** | Image Optimization | 1.5 days | 🟡 Medium | Install EWWW (free), configure lossless compression + WebP, bulk optimize existing library |
| **4** | WordPress CDN Plugin | 1 day | 🟡 Medium | Configure URL rewriting (WP Rocket/W3TC/CDN Enabler), verify src attributes |
| **5** | CDN Caching & Security | 1 day | 🟠 High | Cache TTLs, hotlinking protection, HTTPS enforcement, TLS 1.2+, original image protection |
| **6** | Media Migration Script | 2 days | 🔴 Critical | Write cache warming + DB URL rewriting scripts, test on staging |
| **7** | Run Migration on Production | 0.5 day | 🔴 Critical | Execute migration during low-traffic window, verify, monitor |
| **8** | Analytics & Monitoring | 0.5 day | ✅ Safe | Install Site Kit, Query Monitor, WP Debugging, set up cron alerts |
| **9** | Staging QA & Performance | 2 days | ✅ Safe | Run 25-point QA checklist, benchmark, broken image scan |
| **10** | Deploy to Production | 1 day (+48h monitor) | 🔴 Critical | Production deploy, 48h monitoring, final benchmark, sign-off |
| | **Task 2 Total** | **11.5 days** | | |

### Combined Timeline

```
Week 1:  Task 1 Steps 1-4    (Audit, Page Cache, Redis, Browser Cache)
Week 2:  Task 1 Steps 5-8    (Minification, FastCGI, Staging QA, Deploy)
Week 3:  Task 2 Steps 1-4    (Media Audit, CDN Setup, Image Optimization, CDN Plugin)
Week 4:  Task 2 Steps 5-8    (Security, Migration Script, Production Migration, Monitoring)
Week 5:  Task 2 Steps 9-10   (Staging QA, Production Deploy + 48h monitoring)
Buffer:  ~4 days              (Unexpected issues, DNS propagation delays, plugin conflicts)

Total:   ~25 business days (within 30-day budget)
```

> **⚠️ The buffer days are important.** DNS propagation can take 24-48h. Plugin conflicts may need debugging. EWWW bulk optimization on 10,000+ images can take a full day. Don't schedule back-to-back steps without buffer.

---

## 🚨 High-Risk Steps — Read This Before You Start

> **Junior note:** Not all steps carry equal risk. Some steps, if done wrong, can take down the entire site or cause permanent data loss. Below is a ranked list of the most dangerous steps so you know where to be extra careful.

### 🔴 Severity: Can Take Down The Entire Site

| Step | What Can Go Wrong | Blast Radius | How to Protect Yourself |
|------|-------------------|--------------|------------------------|
| **Task 1 Step 6 — Nginx FastCGI Cache** | A single typo in `nginx.conf` (wrong bracket, bad path, invalid directive) → `nginx -t` fails → if you skip the test and run `systemctl restart nginx` → **entire site goes offline instantly for all users**. Wrong `fastcgi_pass` socket path → every page returns **502 Bad Gateway**. Wrong bypass rules → logged-in admins see cached public pages (or worse, User A sees User B's admin dashboard). | 🔴 **Full site outage** | 1. **ALWAYS** run `sudo nginx -t` before reloading — never skip this. 2. Use `reload` not `restart` (reload = zero downtime on success, no-op on failure). 3. Keep an SSH session open on a second terminal so you can revert if something goes wrong. 4. Test on staging with real traffic patterns first. |
| **Task 1 Step 8 — Deploy Caching to Production** | Deploying untested Nginx config to production. Forgetting to create the cache directory (`/var/run/nginx-cache/`). Different PHP version on prod vs staging (wrong socket path). | 🔴 **Full site outage** | 1. Copy the exact config tested on staging — do not retype. 2. Run `nginx -t` on production before reload. 3. Have the old config backed up so you can revert in seconds: `cp yourdomain.conf yourdomain.conf.bak` |
| **Task 2 Step 6 — CDN Migration Script (Database)** | The migration script does **bulk SQL REPLACE on your entire database** (`wp_posts`, `wp_postmeta`, `wp_options`). If you put the wrong origin URL or CDN URL → every image URL in every post is permanently corrupted. If the script crashes mid-way → half your database has CDN URLs, half has origin URLs — a messy split state. If you accidentally modify `siteurl` or `home` options → **WordPress can't boot at all** (white screen of death). | 🔴 **Data loss / corruption** | 1. **ALWAYS `wp db export` before running** — this is your undo button. 2. Run on staging first and verify with the count queries shown in the doc. 3. The script excludes `siteurl`, `home`, `active_plugins`, `template`, `stylesheet` from replacement — do NOT remove those safeguards. 4. Test the restore process: `wp db import backup.sql` — make sure you know how to roll back before you need to. |
| **Task 2 Step 7 — Production CDN Migration** | Same as Step 6 but on production with real data and real users. Running during peak traffic can cause slow responses. Maintenance mode may be forgotten (site stays in maintenance). | 🔴 **Data loss + downtime** | 1. Schedule during lowest traffic window (2-5 AM). 2. Set a phone alarm to disable maintenance mode. 3. Have a second person verify the site is back up. |

### 🟠 Severity: Can Break Parts of The Site

| Step | What Can Go Wrong | Blast Radius | How to Protect Yourself |
|------|-------------------|--------------|------------------------|
| **Task 1 Step 3 — Redis Object Cache** | If Redis runs out of memory → WordPress throws errors or becomes extremely slow (every request waits for Redis timeout). If `WP_REDIS_GRACEFUL` is not set and Redis crashes → site hangs for 1-2 seconds per page load (timeout on every request × every visitor). Wrong Redis prefix on multisite → **sites see each other's cached data** (User on Site A sees Site B content). | 🟠 **Degraded performance or data leakage** | 1. Set `maxmemory 256mb` + `allkeys-lru` in Redis config. 2. Add `WP_REDIS_GRACEFUL = true` in wp-config.php (already added in our corrected docs). 3. For multisite, use unique `WP_REDIS_PREFIX` per site. 4. Monitor with `redis-cli info memory`. |
| **Task 1 Step 5 — Minification & JS Defer** | Minifying/deferring JavaScript is the #1 cause of "caching broke my site" reports. Defer jQuery → **every jQuery-dependent plugin breaks** (broken sliders, broken forms, broken popups). Minify JS from page builders (Elementor, Divi) → **broken page layouts, missing sections**. CSS concatenation can break relative paths in background images. | 🟠 **Broken site functionality** | 1. Enable ONE feature at a time (first HTML minify, then CSS minify, then JS minify, then defer). 2. After each change, test: home page, a blog post, contact form submission, WooCommerce checkout flow. 3. Check browser console for red errors after each change. 4. Know the rollback: just deactivate the plugin setting. |
| **Task 1 Step 4 — Browser Caching (Nginx Config)** | Incorrect `location` regex in Nginx can accidentally match PHP files → PHP files get cached and served as static files → **security risk** (source code exposed) or all pages return the same cached PHP output. Setting overly long `max-age` without cache-busting → after a deploy, users see old CSS/JS for days (stale assets). | 🟠 **Security risk or stale content** | 1. Be precise with location regexes — only match known static extensions. 2. Always pair long `max-age` with version-based cache-busting (filename hash or query string). 3. Run `nginx -t` after every config change. |
| **Task 2 Step 2 — CDN DNS Setup** | Wrong CNAME record → CDN subdomain (`cdn.yourdomain.com`) doesn't resolve → **all images broken site-wide** (every `<img>` tag fails). Accidentally changing the main domain's DNS record instead of the `cdn` subdomain → **entire site goes offline** until DNS propagates back (up to 48 hours). SSL not provisioned before going live → mixed content warnings, broken images in HTTPS pages. | 🟠 **All images broken or full outage** | 1. Only add/modify the `cdn` CNAME record — do NOT touch the main domain's A record. 2. Use short TTL (300s) during setup so mistakes propagate out quickly. 3. Wait for SSL certificate to be issued before configuring WordPress to use CDN URLs. 4. Test with `curl` before changing any WordPress settings. |
| **Task 2 Step 5 — CDN Hotlinking Protection** | Misconfigured allowed referrers → your own site gets blocked from loading its own images. Forgetting to allow empty referrer → **direct image links (shared on email, Slack, etc.) return 403 Forbidden**. Too aggressive blocking → social media preview cards (Facebook, Twitter) can't fetch your images for link previews. | 🟠 **Images broken for real users** | 1. Always allow your own domain AND `www.` variant AND staging domain. 2. Allow empty referrer (direct access) — needed for email clients, chat apps, social sharing. 3. Test from an incognito browser after enabling. |

### 🟡 Severity: Recoverable Issues

| Step | What Can Go Wrong | How to Protect Yourself |
|------|-------------------|------------------------|
| **Task 1 Step 2 — Page Caching Plugin** | Caching WooCommerce cart/checkout pages → customers see other users' cart contents (privacy violation). Caching pages with nonces → forms stop working after cache TTL (Contact Form 7 submissions fail with "expired session" errors). | Always verify exclusion rules. Test form submissions and WooCommerce checkout after enabling. |
| **Task 2 Step 3 — Image Optimization** | Bulk compressing 10,000+ images with a cloud service → API rate limits hit → incomplete optimization. Lossy compression set too aggressively → visible quality degradation on hero images / product photos. `wp media regenerate` on a large library → takes hours, high CPU, can be interrupted and leave partial thumbnails. | Use "lossless" or "normal" compression (not aggressive). Run `wp media regenerate` during off-hours with `--batch=100`. |
| **Task 2 Step 4 — CDN Plugin Config** | CDN Enabler rewrites ALL URLs including those that shouldn't be on CDN (like dynamic API endpoints). "Enable relative paths" can cause URL resolution issues on nested pages. | Test the plugin on staging first. Review page source to confirm only `/wp-content/uploads/` and `/wp-includes/` are rewritten. |

### ✅ Safe Steps (Low Risk)

| Step | Why It's Safe |
|------|--------------|
| Task 1 Step 1 — Audit | Read-only. You're just checking the current state. |
| Task 1 Step 7 — Staging QA | You're on staging. Break things freely — that's what staging is for. |
| Task 2 Step 1 — Media Audit | Read-only file system and database queries. |
| Task 2 Step 8 — Analytics & Monitoring | Adding monitoring can't break anything — it's observational. |
| Task 2 Step 9 — Staging QA | On staging. Safe to test destructively. |

---

## 🛡️ Universal Safety Rules (Apply to EVERY Step)

> 1. **Never make changes directly on production.** Always staging first.
> 2. **Always have a rollback plan before starting.** Know exactly which command undoes your change.
> 3. **Back up the database before any script that writes to it.** `wp db export backup.sql` takes seconds and saves hours.
> 4. **Back up Nginx config before editing.** `cp yourdomain.conf yourdomain.conf.bak`
> 5. **`nginx -t` before every reload.** No exceptions. Ever.
> 6. **One change at a time.** If you enable 5 things at once and the site breaks, you don't know which one did it.
> 7. **Keep an SSH session open** while making server changes — if the web interface goes down, you still have access.

---

## 🔗 Quick Links

### Task 1 — WordPress Caching Configuration
- [Step 1 — Audit Current Setup](task-1-caching/step-01-audit-current-setup.md)
- [Step 2 — Install & Configure Page Caching Plugin](task-1-caching/step-02-page-caching-plugin.md)
- [Step 3 — Configure Object Caching via Redis](task-1-caching/step-03-object-caching-redis.md)
- [Step 4 — Configure Browser Caching](task-1-caching/step-04-browser-caching.md)
- [Step 5 — Minification & Concatenation](task-1-caching/step-05-minification-concatenation.md)
- [Step 6 — Server-Level Caching Integration](task-1-caching/step-06-server-level-caching.md)
- [Step 7 — Staging Testing & QA](task-1-caching/step-07-staging-testing-qa.md)
- [Step 8 — Deploy to Production & Monitor](task-1-caching/step-08-deploy-production.md)

### Task 2 — CDN Integration for Image Delivery
- [Step 1 — Audit Current Media Library](task-2-cdn/step-01-audit-media-library.md)
- [Step 2 — Select & Set Up CDN Provider](task-2-cdn/step-02-cdn-provider-setup.md)
- [Step 3 — Configure Image Optimization Pipeline](task-2-cdn/step-03-image-optimization-pipeline.md)
- [Step 4 — Install & Configure WordPress CDN Plugin](task-2-cdn/step-04-wordpress-cdn-plugin.md)
- [Step 5 — Configure CDN Caching & Security Policies](task-2-cdn/step-05-cdn-caching-security.md)
- [Step 6 — Write & Test Media Migration Script](task-2-cdn/step-06-media-migration-script.md)
- [Step 7 — Migrate Existing Images to CDN](task-2-cdn/step-07-migrate-images-to-cdn.md)
- [Step 8 — Integration with Analytics & Monitoring](task-2-cdn/step-08-analytics-monitoring.md)
- [Step 9 — Staging QA & Performance Testing](task-2-cdn/step-09-staging-qa-performance.md)
- [Step 10 — Deploy to Production & Monitor](task-2-cdn/step-10-deploy-production.md)

# wordpress-cache-cdn-optimization
