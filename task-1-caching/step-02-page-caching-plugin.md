# Task 1 — Step 2: Install & Configure Page Caching Plugin

> **Estimated Duration:** 1 day
> **Goal:** Enable static HTML page caching to dramatically reduce server load and improve TTFB.

---

## 🎯 Objectives
- Choose the right page caching plugin for your project
- Install and configure it on staging
- Set correct cache TTL for different page types
- Enable automatic cache clearing on content updates

---

## 📌 Which Plugin Should You Choose?

> **Junior note — What is page caching?**
> Every time someone visits a WordPress page, PHP runs code, queries the database, assembles HTML, and sends it to the browser. This takes 300–800ms per visit. Page caching saves the finished HTML to a file the FIRST time. On the next visit, WordPress serves the saved file instantly (~10–50ms) without running any PHP or database queries. It's like printing a document once and photocopying it instead of rewriting it from scratch each time.

### Decision Matrix

| Criteria | WP Rocket | W3 Total Cache |
|----------|:---------:|:--------------:|
| **Price** | ~$59/year (commercial) | Free |
| **Setup difficulty** | 🟢 Easy (5 minutes) | 🟠 Medium-Hard (30+ minutes) |
| **Risk of misconfiguration** | 🟢 Low (safe defaults) | 🔴 High (many options, easy to break) |
| **Page caching** | ✅ Built-in | ✅ Built-in |
| **Object caching (Redis)** | ❌ No (use Redis Object Cache plugin) | ✅ Built-in |
| **Minification / Defer JS** | ✅ Built-in | ✅ Built-in |
| **CDN integration** | ✅ Built-in (Step 2-CDN) | ✅ Built-in |
| **Browser caching headers** | ✅ Automatic | ✅ Manual config |
| **Cache preloading** | ✅ Automatic sitemap crawl | ✅ Manual "prime cache" |
| **Auto purge on content update** | ✅ Default, zero config | ⚠️ Needs manual setup |
| **WooCommerce compatibility** | ✅ Excellent (auto-excludes cart/checkout) | ⚠️ Manual exclusion rules needed |
| **Multisite support** | ✅ Per-site first (see below) | ✅ Network-wide first (see below) |
| **Nginx FastCGI Cache integration** | ❌ No (use Nginx Helper plugin) | ✅ Built-in support |
| **Plugin conflict risk** | 🟢 Low | 🟠 Medium (many overlapping settings) |
| **Support** | ✅ Paid support team | ❌ Community forums only |
| **Documentation quality** | ✅ Excellent | ⚠️ Outdated in places |

### When to Choose WP Rocket

✅ **Choose WP Rocket if:**
- Budget allows ~$59/year — it pays for itself in time saved
- You want the fastest, safest setup with minimal risk
- You're running WooCommerce (automatic cart/checkout exclusion)
- You want one plugin to handle page caching + minification + CDN + lazy loading
- You don't want to spend time tuning 50+ settings
- You need reliable commercial support

> 💡 WP Rocket is the only caching plugin that works well **out of the box** with zero configuration. Activate it and you already get ~70% of the performance benefit.

### When to Choose W3 Total Cache

✅ **Choose W3 Total Cache if:**
- Budget is $0 (strict constraint)
- You need **object caching + page caching + CDN in a single plugin** (W3TC does all three)
- You want granular control over every caching parameter
- You're comfortable reading documentation and debugging config issues
- You're on a server where you need Nginx FastCGI Cache integration from the plugin itself
- You enjoy fine-tuning and don't mind spending more time on setup

> ⚠️ W3 Total Cache has more settings, which means more ways to break things. Budget extra time for testing after every change.

### Multisite: "Per-Site First" vs "Network-Wide First" — What Does This Mean?

> 💡 **Junior note:** Both plugins fully support WordPress Multisite. The difference is their **default management approach** — who controls the settings by default.

| | WP Rocket ("Per-Site First") | W3 Total Cache ("Network-Wide First") |
|--|------------------------------|---------------------------------------|
| **Default approach** | Each sub-site manages its own caching rules independently | Network Admin sets ONE set of rules that apply to ALL sub-sites |
| **Who controls settings?** | Sub-site admins (Site A admin, Site B admin, etc.) | Network Admin (Super Admin) |
| **Can you override?** | Yes — Network Admin can restrict sub-site access | Yes — Sub-site admins can override network defaults for their own site |
| **Example** | Site A: TTL = 4h, exclude `/members/` <br> Site B: TTL = 12h, exclude `/shop/` | All sites: TTL = 8h, exclude `/cart/` — then Site B overrides TTL to 12h |
| **Best for** | Sub-sites with **different needs** (blog vs shop vs forum) | Sub-sites that are **mostly identical** (franchise locations, multi-language) |
| **Risk** | A sub-site admin could misconfigure their own cache | One bad network-level setting breaks ALL sub-sites at once |

**In plain English:**
- **WP Rocket** says: "Each site, you figure out your own caching." The Network Admin can step in and lock things down if needed.
- **W3 Total Cache** says: "I'll set the rules for everyone." Individual sites can ask for exceptions.

Both approaches are valid — choose based on how your Multisite network is structured. See the detailed Multisite sections in [Step 2A (WP Rocket)](step-02a-wp-rocket.md#2a8--multisite-configuration) and [Step 2B (W3TC)](step-02b-w3-total-cache.md#2b8--multisite-configuration).

### Third Option: LiteSpeed Cache (Honorable Mention)

If your server runs **LiteSpeed Web Server** (not Nginx, not Apache), use **LiteSpeed Cache** — it's free, extremely fast, and deeply integrated with the server. But it **only works on LiteSpeed servers**. If you're on Nginx (which this guide assumes), skip it.

---

## 📂 Plugin-Specific Guides

Choose your plugin and follow the dedicated guide:

| Guide | File |
|-------|------|
| **WP Rocket** (Recommended) | [step-02a-wp-rocket.md](step-02a-wp-rocket.md) |
| **W3 Total Cache** (Free alternative) | [step-02b-w3-total-cache.md](step-02b-w3-total-cache.md) |

> ⚠️ **Only install ONE caching plugin.** Running two caching plugins simultaneously causes conflicts — double caching, broken headers, and cache files that never clear. If you switch plugins, fully deactivate and delete the old one first.

---

## ✅ Validation Checklist (Both Plugins)

After configuring whichever plugin you chose, verify ALL of these:

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Plugin installed and active on staging | `wp plugin list --status=active` shows the plugin |
| 2 | Cache files are being created | Check `/wp-content/cache/` directory for HTML files |
| 3 | Home page cache TTL = 4 hours | Inspect `Cache-Control` header: `curl -I https://staging.yourdomain.com` |
| 4 | Internal pages TTL = 8 hours | Inspect headers on an inner page |
| 5 | Cache is cleared on post update | Edit a post → save → visit the page → content should be updated |
| 6 | Logged-in users bypass cache | Log in → visit a page → confirm you see live content, not cached |
| 7 | Cart/Checkout pages are not cached | Visit `/cart/` → confirm no `X-Cache: HIT` header |
| 8 | Mobile caching works | Chrome DevTools → Toggle mobile → reload → cached version loads |
| 9 | RSS feeds are cached | `curl -I https://staging.yourdomain.com/feed/` → cache headers present |
| 10 | No other caching plugin active | `wp plugin list --status=active` shows ONLY your chosen caching plugin |

---

> ⬅️ [Previous: Step 1 — Audit](step-01-audit-current-setup.md) &nbsp;|&nbsp; ➡️ [Next: Step 3 — Object Caching (Redis)](step-03-object-caching-redis.md)

