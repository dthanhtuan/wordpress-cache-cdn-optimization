# 📊 What Does The Plugin Handle vs What You Must Do Manually

> **Purpose:** A quick-reference matrix showing which tasks are handled automatically by your caching plugin (WP Rocket or W3 Total Cache) and which require manual server/code configuration.
>
> **How to read:** ✅ Plugin handles it = just enable the toggle. 🔧 Manual = you must configure the server or write code. ⚠️ Partial = plugin does some of it, but you need to do the rest.

---

## Task 1 — WordPress Caching Configuration

| Step | Feature | WP Rocket | W3 Total Cache | Manual (Server/Code) | Notes |
|------|---------|-----------|----------------|---------------------|-------|
| **1** | **Audit current setup** | ❌ | ❌ | 🔧 You do it | Run commands to check PHP version, existing plugins, server type. Read-only. |
| **2** | **Page caching** | ✅ Auto on activation | ✅ Enable toggle | ❌ Not needed | Plugin generates cached HTML files. |
| **2** | Page cache exclusions (cart, checkout) | ✅ Auto for WooCommerce | 🔧 **Manual — CRITICAL** | ❌ | W3TC does NOT auto-detect WooCommerce. You must add exclusions yourself. |
| **2** | Cache preloading | ✅ Auto (sitemap crawl) | ⚠️ Enable + configure | ❌ | W3TC needs sitemap URL set manually. |
| **2** | Auto-purge on content update | ✅ Auto, zero config | ⚠️ Enable checkboxes | ❌ | W3TC needs purge policy checkboxes checked manually. |
| **3** | **Object caching (Redis)** | ❌ Not included | ⚠️ Built-in but use separate plugin | 🔧 Install Redis server | Both need **Redis Object Cache** plugin + Redis server installed. WP Rocket has no object cache; W3TC has one but the dedicated plugin is better. |
| **4** | **Browser caching headers** | ⚠️ Apache only (`.htaccess`) | ⚠️ Apache only (`.htaccess`) | 🔧 **Nginx: MUST do manually** | Plugins write `.htaccess` which Nginx ignores. On Nginx, you MUST configure headers in `nginx.conf` yourself. |
| **4** | Gzip / Brotli compression | ⚠️ Apache only | ⚠️ Apache only | 🔧 **Nginx: MUST do manually** | Same as above — plugin can only set this for Apache. |
| **5** | **HTML/CSS/JS minification** | ✅ Toggle | ✅ Toggle | ❌ Not needed | Just enable in plugin settings. |
| **5** | JS defer / delay | ✅ Built-in | ❌ Not built-in | 🔧 Install Autoptimize | W3TC users need a separate free plugin for JS defer. |
| **5** | CSS/JS concatenation | ✅ Toggle (skip on HTTP/2) | ✅ Toggle (skip on HTTP/2) | ❌ | Both have it, but you should **not enable it** on HTTP/2 servers. |
| **6** | **Nginx FastCGI Cache** | ❌ Not supported | ⚠️ Partial integration | 🔧 **MUST do manually** | This is server-level config. No plugin can create the Nginx cache zone for you. You must edit `nginx.conf`. Both plugins can help with cache purging via Nginx Helper plugin. |
| **6** | wp-cron replacement | ❌ | ❌ | 🔧 Server crontab | Must disable WP cron and set up a real server cron job. No plugin does this. |
| **7** | **Staging QA** | ❌ | ❌ | 🔧 You do it | Manual testing on staging. |
| **8** | **Deploy to production** | ❌ | ❌ | 🔧 You do it | Copy configs, run checklists. |

---

## Task 2 — CDN Integration for Image Delivery

| Step | Feature | WP Rocket | W3 Total Cache | Manual (Server/Code) | Notes |
|------|---------|-----------|----------------|---------------------|-------|
| **1** | **Audit media library** | ❌ | ❌ | 🔧 You do it | Run WP-CLI commands to count images, check sizes. |
| **2** | **CDN provider setup** | ❌ | ❌ | 🔧 You do it | Create account, configure pull zone, DNS CNAME. No plugin does this. |
| **3** | **CDN URL rewriting** | ✅ Built-in CDN settings | ✅ Built-in CDN settings | ❌ Not needed | Both plugins can rewrite `/wp-content/uploads/` URLs to CDN domain. Or use CDN Enabler (free standalone plugin). |
| **4** | **Image optimization** | ⚠️ Paid add-on (Imagify) | ❌ | 🔧 Install EWWW Image Optimizer (free local mode) | WP Rocket team makes Imagify but it's paid. EWWW free mode uses server-side tools (jpegoptim, optipng, cwebp) — unlimited, no API costs. W3TC has no image optimization. |
| **5** | **CDN caching rules** | ❌ | ❌ | 🔧 Configure in CDN dashboard | Cache TTL, cache-control headers, edge rules — all set in BunnyCDN / Cloudflare dashboard. |
| **5** | Image privacy / access protection | ❌ | ❌ | 🔧 Nginx config + PHP auth | No plugin handles this. Requires custom Nginx rules and PHP auth gate. |
| **5** | CDN token authentication | ❌ | ❌ | 🔧 CDN dashboard + code | URL signing for protected content. Configure in CDN provider. |
| **6** | **Media migration script** | ❌ | ❌ | 🔧 **Custom script (high risk)** | Bulk database URL replacement. No plugin safely handles this at scale. |
| **7** | **Production CDN migration** | ❌ | ❌ | 🔧 You do it | Run migration script on production. |
| **8** | **Analytics & monitoring** | ❌ | ❌ | 🔧 You do it | GA4 setup, monitoring scripts, log rotation. |
| **9** | **Staging QA** | ❌ | ❌ | 🔧 You do it | Manual testing. |
| **10** | **Deploy to production** | ❌ | ❌ | 🔧 You do it | Final deployment and monitoring. |

---

## 📊 Summary: Plugin Coverage by Task

### Task 1 — Caching

```
What the plugin handles (just enable toggles):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Page caching (Step 2)
✅ Page cache exclusions — WP Rocket only (Step 2)
✅ Cache preloading (Step 2)
✅ Auto-purge on content update (Step 2)
✅ HTML / CSS / JS minification (Step 5)
✅ JS defer — WP Rocket only (Step 5)
✅ CDN URL rewriting (used in Task 2 Step 4)

What YOU must do manually:
━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 Audit (Step 1)
🔧 Redis server install + config (Step 3)
🔧 Browser caching on Nginx (Step 4) ← plugin can't help on Nginx
🔧 Gzip/Brotli on Nginx (Step 4) ← plugin can't help on Nginx
🔧 Nginx FastCGI Cache (Step 6) ← server config, no plugin can do this
🔧 wp-cron → server cron (Step 6)
🔧 Staging QA (Step 7)
🔧 Production deploy (Step 8)
```

### Task 2 — CDN

```
What the plugin handles:
━━━━━━━━━━━━━━━━━━━━━━━━
✅ CDN URL rewriting (Step 4)

Everything else is manual:
━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 CDN provider setup, DNS, SSL (Step 2)
🔧 Image optimization — EWWW free plugin (Step 3)
🔧 CDN caching rules, security, access protection (Step 5)
🔧 Database migration script (Step 6) — highest risk step
🔧 Production migration (Step 7)
🔧 Analytics & monitoring (Step 8)
🔧 All QA and deployment (Steps 9-10)
```

---

## 🎯 The Big Picture

> 💡 **Junior note:** The caching plugin (WP Rocket or W3TC) handles roughly **40% of Task 1** and **10% of Task 2**. The rest is server configuration, CDN dashboard settings, custom scripts, and manual testing.
>
> **Don't assume "install plugin = done."** The plugin is one tool in a much larger optimization stack:

```
┌─────────────────────────────────────────────────────────┐
│                    FULL CACHING STACK                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Browser Cache ──── Nginx config (manual)                │
│       ↓                                                  │
│  CDN Edge Cache ─── CDN dashboard (manual)               │
│       ↓                                                  │
│  Nginx FastCGI ──── Nginx config (manual)                │
│       ↓                                                  │
│  Page Cache ─────── Plugin handles ✅                    │
│       ↓                                                  │
│  Object Cache ───── Redis server (manual) + Plugin       │
│       ↓                                                  │
│  MySQL Database                                          │
│                                                          │
│  Minification ───── Plugin handles ✅                    │
│  JS Defer ────────── Plugin handles ✅ (WP Rocket)       │
│  CDN URL Rewrite ── Plugin handles ✅                    │
│  Image Optimize ─── Separate plugin/service              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**The plugin sits in the MIDDLE of the stack.** The layers above it (browser, CDN, Nginx) and below it (Redis, MySQL) all need manual configuration.

---

## ⚡ Quick Decision: Which Plugin + What Manual Work?

### If you choose WP Rocket ($59/year):
```
Plugin auto-handles:     Page cache, preload, purge, WooCommerce exclusions,
                         minification, JS defer, browser cache (Apache only),
                         CDN URL rewriting

You still must manually:  Redis, Nginx browser cache, Nginx FastCGI cache,
                         Gzip/Brotli, wp-cron, CDN provider setup, image
                         optimization (EWWW), migration script, monitoring

Manual effort saved:     ~2 days (auto exclusions, auto preload, auto defer)
```

### If you choose W3 Total Cache (free):
```
Plugin handles:          Page cache, minification, browser cache (Apache only),
                         CDN URL rewriting, partial Nginx integration

You must also manually:  Redis, Nginx browser cache, Nginx FastCGI cache,
                         Gzip/Brotli, wp-cron, WooCommerce exclusions,
                         purge policy, preload config, JS defer (Autoptimize),
                         CDN provider setup, image optimization, migration
                         script, monitoring

Manual effort saved:     ~0.5 days (page cache + minify toggles)
```

> 💡 **Bottom line:** WP Rocket saves you roughly 1.5 extra days of configuration compared to W3TC, mainly from auto-WooCommerce exclusions, zero-config purge, and built-in JS defer.

---

> 📂 [Back to README](README.md)
