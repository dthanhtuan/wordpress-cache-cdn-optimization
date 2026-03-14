# Task 1 — Step 2A: WP Rocket — Full Installation & Configuration Guide

> **Estimated Duration:** 0.5 day
> **Goal:** Install and fully configure WP Rocket on staging for page caching, auto-purge, mobile caching, and cache exclusions.
> **Cost:** ~$59/year (single site) or ~$259/year (unlimited sites)
> **Purchase:** [https://wp-rocket.me](https://wp-rocket.me)

---

## 📌 Why Choose WP Rocket?

WP Rocket is a **commercial (paid) caching plugin** built for WordPress. It's the most popular premium caching plugin, used on 4M+ websites.

### Strengths
- **Zero-config performance:** Activate it and you immediately get page caching, browser caching, GZIP compression, and cache preloading — no settings to touch
- **Safe defaults:** WP Rocket's defaults are tuned by caching experts. Unlike W3 Total Cache, you can't easily break your site by clicking the wrong checkbox
- **WooCommerce auto-detection:** Automatically detects WooCommerce and excludes `/cart/`, `/checkout/`, `/my-account/` — you don't have to remember
- **One-click features:** Minification, defer JS, lazy loading, and CDN are all toggle switches
- **Reliable auto-purge:** When you save/update a post, WP Rocket automatically purges that post's cache AND related pages (home, category, tag archives). Zero config needed
- **Cache preloading:** Automatically crawls your sitemap and pre-builds cache files, so the first visitor never gets an uncached (slow) page
- **Commercial support:** Paid support team responds within 24 hours

### Weaknesses
- **Costs money** ($59/year minimum). No free tier
- **No object caching built-in** — you still need a separate plugin for Redis/Memcached (see Step 3)
- **No Nginx FastCGI Cache integration** — you'll use Nginx Helper plugin for that (see Step 6)
- **Can't install via WP-CLI** from the plugin directory — must upload the .zip manually
- **Slight overlap with Step 6:** WP Rocket's page caching and Nginx FastCGI Cache both cache HTML. When you enable FastCGI Cache in Step 6, Nginx serves cached pages BEFORE WP Rocket is even invoked. WP Rocket then acts as a fallback for any requests that miss the Nginx cache. This is fine — they don't conflict

### When NOT to Choose WP Rocket
- ❌ Budget is $0 with no flexibility → use W3 Total Cache instead
- ❌ You need object caching + page caching in a single plugin → W3 Total Cache does both
- ❌ Your server runs LiteSpeed → use LiteSpeed Cache (free and faster on that server)

---

## 🔧 Detailed Steps

### 2A.1 — Purchase and Download

1. Go to [https://wp-rocket.me](https://wp-rocket.me)
2. Choose your plan:
   - **Single:** $59/year (1 website)
   - **Plus:** $119/year (3 websites)
   - **Infinite:** $259/year (unlimited websites)
3. Download the `wp-rocket_X.X.X.zip` file from your account dashboard

---

### 2A.2 — Install on Staging First

> ⚠️ **ALWAYS install on staging first.** Never install a caching plugin directly on production — if something goes wrong, you want it to break on staging, not in front of real users.

```bash
# Method 1: Upload via WordPress Admin
# Dashboard → Plugins → Add New → Upload Plugin → select wp-rocket_X.X.X.zip → Install Now → Activate

# Method 2: Upload via SSH/SFTP
# Upload wp-rocket_X.X.X.zip to /var/www/html/wp-content/plugins/
unzip wp-rocket_X.X.X.zip -d /var/www/html/wp-content/plugins/
wp plugin activate wp-rocket
```

After activation, WP Rocket **immediately starts working** with sensible defaults. You'll already see:
- Page caching enabled
- GZIP compression enabled
- Browser caching headers added
- Cache preloading started (crawls your sitemap in the background)

> 💡 **Junior note:** WP Rocket is the only caching plugin that starts improving performance the second you activate it. The steps below are about fine-tuning, not making it work.

---

### 2A.3 — Configure Cache Settings

Navigate to **Settings → WP Rocket → Cache** tab:

```
✅ Enable caching for mobile devices
   → Serves cached pages to mobile visitors too (most themes are responsive, so this is safe)

✅ Separate cache files for mobile devices
   → ONLY enable this if your theme serves DIFFERENT HTML to mobile vs desktop
   → If your theme is responsive (same HTML, different CSS), leave this OFF
   → Enabling it unnecessarily doubles your cache size for no benefit

✅ Enable caching for logged-in users
   → Creates a separate cache per user role (admin, editor, subscriber)
   → Useful for membership sites where logged-in content is mostly the same
   → ⚠️ Disable this if each user sees unique content (dashboards, personalized pages)

Cache Lifespan: 8 hours (480 minutes)
   → This is how long a cached page lives before WP Rocket regenerates it
   → 8 hours is a good balance for most sites
```

> 💡 **Junior note — What is "Cache Lifespan"?**
> Imagine printing 100 copies of today's newsletter. After 8 hours, you throw them all away and print fresh ones (in case the content changed). That's cache lifespan. Shorter = more fresh but more server work. Longer = faster but content updates are delayed.

---

### 2A.4 — Set Different TTL for Home Page (Custom Code)

The home page typically changes more often (new posts, featured content) than individual pages. Set a shorter TTL for it:

Add to `functions.php` of your active theme or a custom plugin:
```php
/**
 * Custom cache TTL per page type for WP Rocket.
 *
 * Home page: 4 hours (changes more often — new posts, featured content)
 * All other pages: 8 hours (content is more stable)
 *
 * Why not use the same TTL for everything?
 * - Home page shows latest posts — if TTL is 8 hours, a newly published post
 *   won't appear on the home page for up to 8 hours (or until auto-purge fires)
 * - Inner pages (About, Contact, old blog posts) rarely change — longer TTL is fine
 */
add_filter( 'rocket_post_expiration_time', function( $lifespan, $post_id ) {
    // Home page / front page → 4 hours
    if ( $post_id === (int) get_option( 'page_on_front' ) ) {
        return 4 * HOUR_IN_SECONDS;
    }

    // Blog index page → 4 hours (also shows latest posts)
    if ( $post_id === (int) get_option( 'page_for_posts' ) ) {
        return 4 * HOUR_IN_SECONDS;
    }

    // Everything else → 8 hours
    return 8 * HOUR_IN_SECONDS;
}, 10, 2 );
```

---

### 2A.5 — Configure Cache Preloading

Navigate to **Settings → WP Rocket → Preload** tab:

```
✅ Activate Preloading
   → WP Rocket crawls your sitemap and pre-generates cache files for all pages
   → This means the FIRST visitor to any page gets a cached (fast) response

✅ Enable link preloading (prefetch)
   → When a user hovers over a link, WP Rocket starts loading that page in the background
   → By the time they click, the page is already partially loaded — feels instant

Sitemap-based preloading:
   → WP Rocket auto-detects Yoast SEO, Rank Math, or default WP sitemaps
   → If using a custom sitemap, add the URL here
```

> 💡 **Junior note — Why preload?**
> Without preloading, the first visitor after cache expires gets a slow page (cache MISS). With preloading, WP Rocket crawls your site in the background and rebuilds cache before any visitor hits it. Every visitor gets a fast page.

---

### 2A.6 — Configure Cache Exclusions

Navigate to **Settings → WP Rocket → Advanced Rules** tab:

#### Never Cache URLs
Add these URL patterns (one per line):
```
/wp-admin/(.*)
/wp-login.php
```

> 💡 **WooCommerce note:** WP Rocket **automatically detects WooCommerce** and excludes `/cart/`, `/checkout/`, and `/my-account/` pages. You don't need to add them manually. But verify anyway (see validation checklist).

#### Never Cache Cookies
WP Rocket auto-detects `wordpress_logged_in_*` cookies. Add any custom cookies from membership plugins:
```
# If using MemberPress, Restrict Content Pro, or similar:
memberpress_*
rcp_*
```

#### Never Cache User Agents
Leave default unless you have specific bot/crawler requirements.

#### Always Purge URLs
Add URLs that should be purged whenever ANY post is updated:
```
/                    ← Home page
/blog/               ← Blog archive (if separate from home)
/sitemap_index.xml   ← XML sitemap
```

---

### 2A.7 — Configure Auto Cache Clearing (Purge)

WP Rocket automatically purges cache when content changes. **This is enabled by default** with no config needed. Here's what it does:

| Trigger | What Gets Purged |
|---------|-----------------|
| Post saved/updated | That specific post URL |
| New post published | Home page + category + tag archives + RSS feed |
| Comment approved | The post that received the comment |
| Widget updated | Entire site cache (widgets appear on all pages) |
| Theme/plugin updated | Entire site cache |
| WP Rocket settings saved | Entire site cache |

To manually purge:
```bash
# Via WP-CLI
wp rocket clean --confirm           # Purge entire cache
wp rocket clean --post_id=123       # Purge specific post

# Via Admin Bar
# When logged in as admin → top bar → "Clear this cache" (per-page)
# Or: WP Rocket → Clear Cache (entire site)
```

---

### 2A.8 — Multisite Configuration

> Skip this section if you're running a single-site WordPress install.

WP Rocket supports Multisite **out of the box**. Each sub-site gets its own cache directory automatically.

#### WP Rocket's Multisite Approach: "Per-Site First"

WP Rocket defaults to a **per-site management model** — each sub-site admin controls their own caching rules independently. This is the opposite of W3 Total Cache, which defaults to network-wide control.

> 💡 **Junior note — What does "per-site first" mean?**
>
> Imagine you have 3 sub-sites: a Blog, a Shop, and a Forum.
> - **Blog admin** sets TTL = 4 hours (posts change often)
> - **Shop admin** sets TTL = 12 hours, excludes `/cart/` and `/checkout/`
> - **Forum admin** disables page caching entirely (all content is dynamic)
>
> Each admin makes decisions for their own site without affecting others.
> The Network Admin (Super Admin) can step in and restrict access if needed.

**Network Admin → Settings → WP Rocket:**
```
Option 1 (Default — Per-Site):
✅ Allow sub-site admins to manage their own WP Rocket settings
   → Each sub-site admin can configure TTL, exclusions, etc. independently
   → Best for: sub-sites with DIFFERENT needs (blog vs shop vs forum)
   → Risk: a sub-site admin could misconfigure their own cache

Option 2 (Centralized — Network-Wide):
❌ Manage all settings from Network Admin only
   → All sub-sites share the same caching rules
   → Best for: uniform sub-sites (franchise locations, multi-language)
   → Risk: one bad setting affects ALL sub-sites
```

> 💡 **Junior note:** On Multisite, WP Rocket creates separate cache folders per sub-site. Purging one sub-site **never** affects others — this is true regardless of which management model you choose.

**For subdirectory Multisite** (`yourdomain.com/site1/`):
```
Cache directories:
/wp-content/cache/wp-rocket/yourdomain.com/site1/
/wp-content/cache/wp-rocket/yourdomain.com/site2/
```

**For subdomain Multisite** (`site1.yourdomain.com`):
```
Cache directories:
/wp-content/cache/wp-rocket/site1.yourdomain.com/
/wp-content/cache/wp-rocket/site2.yourdomain.com/
```

#### When to Use Which Model

| Scenario | Recommended Model |
|----------|------------------|
| Each sub-site is a different business unit (blog, shop, docs) | ✅ Per-site (let each admin tune their own) |
| All sub-sites are the same product in different languages | ✅ Network-wide (one config for all) |
| Sub-site admins are junior developers | ✅ Network-wide (reduce risk of misconfiguration) |
| Sub-site admins are experienced WordPress users | ✅ Per-site (give them autonomy) |

---

### 2A.9 — Verify WP Rocket is Working

```bash
# Check plugin is active
wp plugin list --status=active | grep wp-rocket

# Check cache files exist
ls -la /var/www/html/wp-content/cache/wp-rocket/

# Visit a page twice and check response headers
curl -I https://staging.yourdomain.com/
# First request — page is being cached
# Second request — look for these WP Rocket indicators:
#   - HTML source contains: <!-- This website is like a Rocket -->
#   - Or: <!-- Cached page generated by WP Rocket -->

# Check home page source for WP Rocket comment (proves caching works)
curl -s https://staging.yourdomain.com/ | tail -5
# Should see: <!-- This website is like a Rocket, isn't it? ... -->
```

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | WP Rocket installed and active | `wp plugin list --status=active \| grep wp-rocket` |
| 2 | Cache files being created | `ls /wp-content/cache/wp-rocket/` shows directories |
| 3 | Cached page has WP Rocket comment | `curl -s https://staging.yourdomain.com/ \| tail -5` shows Rocket comment |
| 4 | Home page TTL = 4 hours | Custom filter in `functions.php` returns `14400` |
| 5 | Inner pages TTL = 8 hours | Default cache lifespan set to 480 minutes |
| 6 | Cache clears on post update | Edit post → save → refresh → content updated |
| 7 | Logged-in users bypass cache | Log in → view source → no Rocket cache comment |
| 8 | Cart/Checkout excluded | Visit `/cart/` → no cache headers, no Rocket comment |
| 9 | Mobile caching works | DevTools → mobile toggle → reload → cached version |
| 10 | Preloading active | WP Rocket → Preload → shows "Preload is running" |
| 11 | No other caching plugin active | `wp plugin list` shows no W3TC, LiteSpeed Cache, etc. |
| 12 | Multisite: each sub-site has own cache dir | Check `/wp-content/cache/wp-rocket/` for per-site folders |

---

## 🚨 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Pages not being cached | Another caching plugin is active | Deactivate ALL other caching plugins |
| Cache not clearing on save | Object cache conflict | Clear object cache: `wp cache flush` |
| `wp-config.php` error on activation | WP Rocket adds constants to wp-config.php | Check file permissions: `chmod 644 wp-config.php` |
| Mobile shows desktop version | "Separate cache for mobile" is off but theme serves different HTML | Enable "Separate cache files for mobile devices" |
| 503 error after activation | Server memory issue with preloading | Disable preloading temporarily, increase PHP memory |

---

> ⬅️ [Back to Step 2 — Plugin Selection](step-02-page-caching-plugin.md) &nbsp;|&nbsp; ➡️ [Next: Step 3 — Object Caching (Redis)](step-03-object-caching-redis.md)
