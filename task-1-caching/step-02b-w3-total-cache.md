# Task 1 — Step 2B: W3 Total Cache — Full Installation & Configuration Guide

> **Estimated Duration:** 1 day
> **Goal:** Install and fully configure W3 Total Cache on staging for page caching, auto-purge, mobile caching, and cache exclusions.
> **Cost:** Free (open source)
> **Plugin page:** [https://wordpress.org/plugins/w3-total-cache/](https://wordpress.org/plugins/w3-total-cache/)

---

## 📌 Why Choose W3 Total Cache?

W3 Total Cache (W3TC) is the **most feature-rich free caching plugin** for WordPress. It's been around since 2009 and powers millions of sites. Unlike WP Rocket, W3TC gives you granular control over every caching layer — but that power comes with complexity.

### Strengths
- **Completely free** — no premium tier required for core features
- **All-in-one:** Page cache + object cache (Redis/Memcached) + browser cache + CDN + minification in a single plugin — you don't need separate plugins for each layer
- **Nginx FastCGI Cache support:** Built-in integration with Nginx server-level caching (WP Rocket doesn't have this)
- **Multiple cache backends:** Disk (basic), Disk (enhanced), Opcode, Memcached, Redis — choose per feature
- **Granular control:** 100+ settings for fine-tuning every caching behavior
- **Page Cache Method: Disk Enhanced** writes pure `.html` files that Nginx can serve without touching PHP at all

### Weaknesses
- **Complex configuration:** 100+ settings means 100+ ways to break your site. A wrong checkbox can cause white screens, broken layouts, or stale content
- **No safe defaults:** Unlike WP Rocket, W3TC doesn't auto-configure anything. You must manually set up every feature
- **WooCommerce needs manual exclusion:** W3TC does NOT auto-detect WooCommerce pages. Forgetting to exclude `/cart/` and `/checkout/` means customers see other users' cart contents (**privacy violation**)
- **Auto-purge requires setup:** Cache doesn't automatically clear on content updates unless you configure it
- **Support is community-only:** Free plugin = no paid support team. You rely on forums and documentation
- **UI is confusing:** Settings are spread across 15+ sub-pages with overlapping options

### When NOT to Choose W3 Total Cache
- ❌ You want a "set and forget" solution → WP Rocket is simpler
- ❌ You're a junior developer doing this for the first time → W3TC's complexity increases risk
- ❌ You're running WooCommerce and don't want to manually configure exclusions → WP Rocket auto-handles this
- ❌ You don't have time to test 100+ settings → WP Rocket works in 5 minutes

---

## 🔧 Detailed Steps

### 2B.1 — Install on Staging First

> ⚠️ **ALWAYS install on staging first.** W3TC modifies `.htaccess`, `wp-config.php`, and creates advanced cache files. If something goes wrong, you want it on staging.

```bash
# Install and activate via WP-CLI
wp plugin install w3-total-cache --activate

# Verify installation
wp plugin list --status=active | grep w3-total-cache
```

Or via WordPress Admin:
```
Dashboard → Plugins → Add New → search "W3 Total Cache" → Install Now → Activate
```

After activation, W3TC adds a **"Performance"** menu to the WordPress admin sidebar. All settings live there.

> ⚠️ **W3TC modifies your `wp-config.php` automatically** on activation, adding:
> ```php
> define('WP_CACHE', true);
> ```
> This is normal and required. If you deactivate W3TC later, remember to remove this line.

---

### 2B.2 — General Settings — Enable Page Cache

Navigate to **Performance → General Settings**:

```
Page Cache:              ✅ Enable
Page Cache Method:       Disk: Enhanced  ← IMPORTANT: choose this one
```

> 💡 **Junior note — What does "Disk: Enhanced" mean?**
>
> W3TC offers several page cache storage methods:
>
> | Method | How It Works | Speed | Best For |
> |--------|-------------|-------|----------|
> | **Disk: Basic** | Saves HTML as PHP files. PHP still runs on every request to serve them. | 🟡 Medium | Shared hosting with no server access |
> | **Disk: Enhanced** | Saves pure `.html` files. Nginx/Apache can serve them directly WITHOUT running PHP at all. | 🟢 **Fastest** | **Any site with server access (use this)** |
> | **Opcode** | Stores in PHP's opcode cache (APCu). Fast but limited capacity. | 🟢 Fast | Small sites with APCu available |
> | **Memcached / Redis** | Stores HTML in memory cache. Fast but uses RAM. | 🟢 Fast | Sites that already use Memcached/Redis |
>
> **Always choose "Disk: Enhanced"** on Nginx. It generates `.html` files in `/wp-content/cache/page_enhanced/` that Nginx serves directly — PHP never executes for cached pages.

---

### 2B.3 — Configure Page Cache Settings

Navigate to **Performance → Page Cache**:

#### General Section
```
Cache home page:                    ✅ Enable
Cache feed(s):                      ✅ Enable
   → RSS feeds are static XML — safe and beneficial to cache

Cache SSL (HTTPS) requests:         ✅ Enable
   → CRITICAL: Your site uses HTTPS. Without this, HTTPS pages won't be cached.

Cache URI with query string:        ❌ Disable
   → ⚠️ RISK: Query strings often represent dynamic/unique content (search results,
   → filtered views, pagination). Caching them can serve wrong content to users.

Cache 404 (not found) pages:        ❌ Disable
   → 404 pages should be dynamic so WordPress can handle redirects and logging
```

#### Cache Lifetime (TTL)
```
Maximum lifetime of cache objects:   28800 seconds  (= 8 hours)
```

> ⚠️ **W3TC does NOT support per-page TTL out of the box** (unlike WP Rocket). The 28800-second lifetime applies to ALL pages equally — home page, blog posts, static pages. To set a shorter TTL for the home page, add custom code (see Step 2B.5).

#### Garbage Collection
```
Garbage collection interval:         3600 seconds (= 1 hour)
   → W3TC checks for expired cache files every hour and deletes them
```

---

### 2B.4 — Configure Cache Exclusions (CRITICAL)

Navigate to **Performance → Page Cache → Advanced**:

#### Never cache the following pages:
```
/wp-admin/
/wp-login.php
```

#### ⚠️ WooCommerce Exclusions (MUST DO MANUALLY)

> 🔴 **CRITICAL RISK:** W3 Total Cache does NOT auto-detect WooCommerce. If you skip this step, customers will see OTHER USERS' cart contents. This is a **privacy violation** and a **trust-destroying bug**.

Add to **"Never cache the following pages"**:
```
/cart/
/checkout/
/my-account/
/shop/\?.*add-to-cart=
```

Add to **"Rejected cookies"** (prevents caching when WooCommerce cart cookies are set):
```
woocommerce_items_in_cart
woocommerce_cart_hash
wp_woocommerce_session_
```

#### Membership Plugin Exclusions
If using MemberPress, Restrict Content Pro, Paid Memberships Pro, etc.:
```
# Add to "Never cache the following pages":
/members-area/
/login/
/register/
/account/

# Add to "Rejected cookies":
memberpress_*
rcp_*
```

#### Contact Form Nonce Exclusions
```
# Forms with nonces can expire if the page is cached too long.
# If forms stop working after enabling caching, add the form pages here:
/contact/
/apply/
```

---

### 2B.5 — Set Shorter TTL for Home Page (Custom Code)

W3TC doesn't support per-page TTL in its settings. Add this custom code to `functions.php`:

```php
/**
 * Custom shorter cache TTL for home page in W3 Total Cache.
 *
 * W3TC sets one global TTL (28800s = 8 hours) for all pages.
 * This filter reduces it to 4 hours for the home page and blog index,
 * since they show latest posts and change more frequently.
 *
 * How it works:
 * - W3TC checks this filter before saving a page to cache
 * - If the current page is home/blog, we return a shorter TTL
 * - W3TC stores the page with that shorter expiry
 */
add_filter( 'w3tc_pgcache_cache_headers', function( $headers ) {
    if ( is_front_page() || is_home() ) {
        $headers['Cache-Control'] = 'public, max-age=14400'; // 4 hours
    }
    return $headers;
});
```

---

### 2B.6 — Configure Auto Cache Clearing (Purge)

Navigate to **Performance → Page Cache → Advanced → Purge Policy**:

```
Purge Policy:
✅ Front page           → purge home page when any post is updated
✅ Post page             → purge the specific post that was updated
✅ Blog feed             → purge RSS feed on new post
✅ Post comments pages   → purge post when a comment is approved
✅ Author pages          → purge author archive when their post changes
✅ Categories            → purge category archive pages
✅ Tags                  → purge tag archive pages
✅ Daily archive         → purge date-based archives
✅ Monthly archive       → purge date-based archives
```

#### Enable Cache Preloading (Prime Cache)
```
✅ Automatically prime the page cache
   → W3TC will crawl your sitemap and pre-build cache after each purge

Prime cache interval:     900 seconds (= 15 minutes)
   → How often W3TC rechecks for pages that need re-caching

Sitemap URL:             https://yourdomain.com/sitemap_index.xml
   → Point this to your Yoast SEO / Rank Math sitemap
```

> 💡 **Junior note — Why is auto-purge important?**
> Without auto-purge, when you publish a new blog post:
> 1. The home page still shows the OLD cached version (without your new post)
> 2. Category pages still show old content
> 3. RSS feed still shows old content
> A visitor won't see your new post until the cache expires (up to 8 hours!)
>
> With auto-purge, publishing a post immediately clears the relevant cached pages, and preloading regenerates them in the background.

---

### 2B.7 — Configure Browser Cache (Bonus — W3TC Handles This Too)

Navigate to **Performance → General Settings**:
```
Browser Cache:           ✅ Enable
```

Navigate to **Performance → Browser Cache**:
```
# General
✅ Set Last-Modified header
✅ Set expires header
✅ Set cache control header
✅ Set entity tag (ETag)

# CSS & JS
Expires header lifetime:  31536000 seconds (= 1 year)
Cache-Control policy:     cache ("public, max-age=31536000")

# HTML & XML
Expires header lifetime:  3600 seconds (= 1 hour)
Cache-Control policy:     cache with max-age

# Media & Other Files
Expires header lifetime:  15778476 seconds (= 6 months)
Cache-Control policy:     cache ("public, max-age=15778476")
```

> ⚠️ **Note:** If you already configured browser caching in Nginx (Step 4), W3TC's browser cache settings may conflict. In that case, **disable W3TC's Browser Cache module** and let Nginx handle it. Nginx is faster at serving headers than PHP.

---

### 2B.8 — Multisite Configuration

> Skip this section if you're running a single-site WordPress install.

W3 Total Cache supports Multisite with **network-wide activation**.

#### W3TC's Multisite Approach: "Network-Wide First"

W3TC defaults to a **network-wide management model** — the Network Admin (Super Admin) sets one set of caching rules that apply to ALL sub-sites. Individual sub-site admins can then override specific settings for their own site.

> 💡 **Junior note — What does "network-wide first" mean?**
>
> Imagine you have 3 sub-sites: English, Spanish, and French versions of the same website.
> - **Network Admin** sets TTL = 8 hours, excludes `/cart/`, enables preloading — for ALL 3 sites at once
> - All 3 sites now share those settings without any per-site configuration
> - If the Spanish site needs a different TTL, its admin can override just that one setting
>
> This is the opposite of WP Rocket, where each sub-site starts with a blank slate and configures everything independently.

```bash
# Network activate (enables for all sub-sites at once)
wp plugin activate w3-total-cache --network
```

Navigate to **Network Admin → Performance → General Settings**:
```
Page Cache:    ✅ Enable (applies to all sub-sites immediately)
```

**Important Multisite behaviors:**
- Each sub-site gets its own cache directory: `/wp-content/cache/page_enhanced/site1.yourdomain.com/`
- **Network admins** set global defaults (TTL, exclusions, purge rules) that all sub-sites inherit
- **Sub-site admins** can override inherited settings for their own site only
- Each sub-site's cache is purged independently (editing a post on Site A doesn't purge Site B)

#### Per-Site Overrides

Once network-activated, sub-site admins see **Performance** in their own admin menu. They can override:
- Cache TTL (lifespan)
- Cache exclusion URLs
- Purge policy checkboxes
- Browser cache settings

Settings they **cannot** override (controlled by Network Admin only):
- Cache method (Disk: Enhanced, Memcached, etc.)
- Nginx rewrite rules
- Plugin activation/deactivation

#### When to Use Which Model

| Scenario | Recommended Approach |
|----------|---------------------|
| All sub-sites are the same product in different languages | ✅ Network-wide defaults, minimal per-site overrides (W3TC's sweet spot) |
| Each sub-site is a different business unit | ⚠️ W3TC works but requires more per-site overrides — WP Rocket may be easier |
| Sub-site admins are junior developers | ✅ Lock settings at network level, don't allow overrides |
| You want maximum consistency across all sites | ✅ Network-wide defaults, disable per-site overrides |

> ⚠️ **RISK:** One bad network-level setting (e.g., wrong cache method, missing exclusion) breaks ALL sub-sites simultaneously. Always test network-level changes on staging first.

> ⚠️ **RISK for subdirectory Multisite:** If using `/site1/`, `/site2/` paths, make sure W3TC's cache file structure separates by path, not just domain. Verify by checking:
> ```bash
> ls /var/www/html/wp-content/cache/page_enhanced/yourdomain.com/site1/
> ls /var/www/html/wp-content/cache/page_enhanced/yourdomain.com/site2/
> ```

---

### 2B.9 — Enable Nginx Compatibility Mode

> This step is **Nginx-specific**. Skip if you're on Apache.

W3TC can generate Nginx rewrite rules for Disk: Enhanced mode. Navigate to:

**Performance → General Settings → Network Performance → Nginx:**
```
✅ Enable Nginx server configuration file path
Path: /etc/nginx/sites-enabled/yourdomain.conf
```

Then go to **Performance → Page Cache → Advanced**:
```
✅ Output Nginx rewrite rules
```

W3TC will display Nginx rules to add to your virtual host config. Copy them:
```bash
# W3TC generates something like this — add to your server {} block:
# location ~ /wp-content/cache/page_enhanced/.*\.html$ {
#     add_header X-Cache "W3TC Disk Enhanced";
# }
```

> ⚠️ **If using Nginx FastCGI Cache (Step 6):** The FastCGI cache is a HIGHER layer than W3TC's Disk Enhanced cache. Nginx FastCGI Cache serves responses before PHP runs. W3TC acts as a fallback for requests that miss the FastCGI cache. They can coexist, but FastCGI Cache does the heavy lifting.

---

### 2B.10 — Verify W3 Total Cache is Working

```bash
# Check plugin is active
wp plugin list --status=active | grep w3-total-cache

# Check cache files exist (after visiting the home page)
curl -s https://staging.yourdomain.com/ > /dev/null
ls -la /var/www/html/wp-content/cache/page_enhanced/

# Check the generated HTML for W3TC signature
curl -s https://staging.yourdomain.com/ | tail -10
# Should see a comment like:
# <!-- Performance optimized by W3 Total Cache ... -->

# Check Cache-Control headers
curl -I https://staging.yourdomain.com/
# Look for: Cache-Control: public, max-age=28800  (or 14400 for home page)

# Test that excluded pages are NOT cached
curl -s https://staging.yourdomain.com/cart/ | grep "W3 Total Cache"
# Should return NOTHING (cart page is not cached)
```

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | W3TC installed and active | `wp plugin list --status=active \| grep w3-total-cache` |
| 2 | Page Cache method = Disk: Enhanced | Performance → General Settings shows "Disk: Enhanced" |
| 3 | Cache files being created | `ls /wp-content/cache/page_enhanced/` shows `.html` files |
| 4 | W3TC comment in page source | `curl -s ... \| tail -10` shows W3TC HTML comment |
| 5 | Home page TTL = 4 hours | `curl -I` shows `max-age=14400` (via custom filter) |
| 6 | Inner pages TTL = 8 hours | `curl -I` on inner page shows `max-age=28800` |
| 7 | Cache clears on post update | Edit post → save → refresh → content updated |
| 8 | Purge policy has all checkboxes | Front page, post, feed, comments, archives all checked |
| 9 | Cache preloading active | Performance → Page Cache → "Automatically prime" = checked |
| 10 | Logged-in users bypass cache | Log in → view source → no W3TC cache comment |
| 11 | `/cart/` excluded | Visit cart → no W3TC comment in source |
| 12 | `/checkout/` excluded | Visit checkout → no W3TC comment in source |
| 13 | `/my-account/` excluded | Visit my-account → no W3TC comment in source |
| 14 | WooCommerce cookies in reject list | "Rejected cookies" contains `woocommerce_items_in_cart` |
| 15 | Mobile caching works | DevTools → mobile toggle → cached version loads |
| 16 | No other caching plugin active | No WP Rocket, LiteSpeed Cache, etc. in plugin list |
| 17 | Multisite: per-site cache dirs | Check `/cache/page_enhanced/` for per-site folders |

---

## 🚨 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| White screen after activation | W3TC failed to write `wp-config.php` or `.htaccess` | Check file permissions: `chmod 644 wp-config.php` and `chmod 644 .htaccess` |
| Pages not being cached | Page Cache not enabled or wrong method selected | Performance → General Settings → verify Page Cache = Disk: Enhanced |
| Stale content after updates | Purge policy not configured | Performance → Page Cache → Advanced → check all purge checkboxes |
| Customer sees another user's cart | WooCommerce pages not excluded | Add `/cart/`, `/checkout/`, `/my-account/` to "Never cache" AND add WooCommerce cookies to "Rejected cookies" |
| CSS/JS broken after enabling minification | W3TC's minify conflicts with theme/plugins | Disable minification in W3TC; use WP Rocket or Autoptimize for minify instead |
| 500 error on some pages | W3TC wrote bad Nginx/Apache rewrite rules | Remove W3TC-generated rules from `.htaccess` or Nginx config |
| "Failed to write to file" errors | Cache directory permissions | `chown -R www-data:www-data /var/www/html/wp-content/cache/` |
| Performance → Page Cache page is blank | PHP memory limit too low for W3TC's admin | Increase PHP memory: `define('WP_MEMORY_LIMIT', '256M');` in `wp-config.php` |

---

## ⚠️ W3TC Settings You Should NOT Touch (Risk of Breaking)

> These W3TC features are powerful but dangerous. Unless you know exactly what you're doing, leave them at defaults:

| Setting | Risk |
|---------|------|
| **Minify → JS/CSS** | W3TC's built-in minifier frequently breaks themes. Use a dedicated tool (Autoptimize) or leave for Step 5. |
| **Database Cache** | Rarely helps and can cause stale query results. MySQL's own query cache is more reliable. |
| **Object Cache** | Configure this separately in Step 3 (Redis). Don't configure both W3TC object cache AND Redis Object Cache plugin — they'll conflict. |
| **Fragment Cache** | Advanced feature for caching parts of pages. Very easy to break dynamic content. Skip unless you're an expert. |
| **Reverse Proxy** | For Varnish setups only. If you're on Nginx, don't touch this. |

---

> ⬅️ [Back to Step 2 — Plugin Selection](step-02-page-caching-plugin.md) &nbsp;|&nbsp; ➡️ [Next: Step 3 — Object Caching (Redis)](step-03-object-caching-redis.md)
