# Task 1 — Step 4: Configure Browser Caching — Server Selection Guide

> **Estimated Duration:** 1 day
> **Goal:** Instruct browsers to cache static assets locally so repeat visitors load pages faster without re-downloading unchanged files.

---

## 🎯 What Is Browser Caching?

When a visitor loads your page, the browser downloads static files — CSS, JavaScript, images, fonts. **Browser caching** tells the browser: "Keep these files locally. Don't download them again until they expire."

Without browser caching:
```
Visit 1: Download style.css (50KB) + logo.png (80KB) + app.js (120KB) = 250KB
Visit 2: Download style.css (50KB) + logo.png (80KB) + app.js (120KB) = 250KB again!
```

With browser caching:
```
Visit 1: Download style.css + logo.png + app.js = 250KB
Visit 2: Browser uses local copies = 0KB transferred → instant load
```

Browser caching is configured on the **web server** — it adds HTTP headers (`Cache-Control`, `Expires`) that tell the browser how long to keep each file type.

---

## 🔀 Do I Need This Step? Can't WP Rocket / W3TC Handle Browser Caching?

**Short answer:** It depends on your web server.

Both WP Rocket and W3 Total Cache have built-in browser caching features. But they work by **writing rules to `.htaccess`** — which is an **Apache-only** mechanism. Nginx completely ignores `.htaccess` files.

| Your Server | Plugin Handles Browser Caching? | Do You Need Step 4? |
|------------|-------------------------------|---------------------|
| **Nginx** | ❌ No — plugins write `.htaccess`, Nginx ignores it | ✅ **Yes, follow Step 4A manually** |
| **Apache** | ✅ Yes — WP Rocket does it automatically, W3TC has a toggle | ⚠️ Optional — verify headers are correct, but plugin handles it |

### On Nginx (This Guide's Default)

WP Rocket enables browser caching on activation — but it writes to `.htaccess`, which Nginx never reads. **Your static files have NO caching headers until you configure Nginx manually.** This is the most common "gotcha" for teams using WP Rocket on Nginx servers.

→ **You MUST follow [Step 4A — Nginx Browser Caching](step-04a-browser-caching-nginx.md)**

### On Apache

- **WP Rocket:** Browser caching is enabled automatically on activation. You can skip Step 4B entirely, but we recommend running the validation checklist to confirm headers are correct.
- **W3 Total Cache:** Enable under **Performance → Browser Cache → ✅ Enable**. Then configure the sub-settings (see Step 4B for recommended values).

→ **Step 4B is optional** — use it to verify or fine-tune what the plugin sets up.

---

## 🔀 Which Web Server Are You Running?

Browser caching config is **100% dependent on your web server**. You only follow ONE guide:

| Web Server | Config File | Guide |
|-----------|------------|-------|
| **Nginx** | `/etc/nginx/sites-enabled/yourdomain.conf` | 👉 [**Step 4A — Nginx Browser Caching**](step-04a-browser-caching-nginx.md) |
| **Apache** | `/var/www/html/.htaccess` | 👉 [**Step 4B — Apache Browser Caching**](step-04b-browser-caching-apache.md) |

> ⚠️ **Do NOT configure both.** They do the same thing in different syntax. Configuring the wrong one does nothing; configuring both (rare edge case with Nginx as reverse proxy in front of Apache) can cause conflicting headers.

### How to Check Which Server You're Running

```bash
# Check Nginx
nginx -v 2>/dev/null && echo "✅ You're on Nginx → use Step 4A"

# Check Apache
apache2 -v 2>/dev/null && echo "✅ You're on Apache → use Step 4B"
apachectl -v 2>/dev/null && echo "✅ You're on Apache → use Step 4B"
```

> 💡 **Junior note:** If you're not sure, ask your hosting provider or system admin. Most modern WordPress setups use **Nginx** (faster for static files). Shared hosting (GoDaddy, Bluehost, SiteGround) typically uses **Apache**. VPS/cloud (DigitalOcean, AWS, Vultr) usually runs **Nginx**.

---

## 📊 Nginx vs Apache for Browser Caching — Quick Comparison

| Feature | Nginx (Step 4A) | Apache (Step 4B) |
|---------|-----------------|-------------------|
| **Config location** | Server config file (requires SSH + root access) | `.htaccess` in WordPress root (can edit via FTP) |
| **Config reload** | Must run `sudo nginx -t && sudo systemctl reload nginx` | Automatic on next request (reads `.htaccess` every time) |
| **Performance** | ✅ Faster — config loaded once into memory | ⚠️ Slower — `.htaccess` parsed on every request |
| **Brotli compression** | ✅ Available (with `ngx_brotli` module) | ⚠️ Available but less common (`mod_brotli`) |
| **Typical hosting** | VPS, cloud, dedicated servers | Shared hosting, cPanel servers |
| **Risk of breaking site** | 🟠 Medium — bad config = Nginx won't reload | 🟡 Low — bad `.htaccess` = 500 error, easy to revert |
| **This guide assumes** | ✅ Primary (Steps 6, CDN docs assume Nginx) | Fallback option |

---

## 📝 Shared Concepts (Both Servers)

Before following your server-specific guide, understand these concepts that apply to both:

### Cache-Busting (Static File Versioning)

When you set `max-age=31536000` (1 year), the browser won't re-download that file for a whole year. But what if you update your CSS? Visitors would see the old version!

**Cache-busting** solves this. WordPress automatically appends a version query string to assets:
```
style.css?ver=1.2.0  ← browser caches this
style.css?ver=1.3.0  ← new URL → browser treats as new file → downloads fresh
```

Ensure your theme uses `wp_enqueue_scripts()` properly:
```php
// In functions.php — good practice (WordPress handles versioning)
wp_enqueue_style(
    'my-theme-style',
    get_stylesheet_uri(),
    [],
    wp_get_theme()->get( 'Version' ) // version from style.css header
);
```

For production deploys, bump the theme version in `style.css`:
```css
/*
Theme Name: Your Theme
Version: 1.2.0   ← bump this on each deploy to bust CSS/JS cache
*/
```

> 💡 If using a build pipeline (webpack, mix, Vite), use **filename hashing** (`style.a3b8f2.css`) instead of query strings for more reliable cache-busting. Some CDNs strip query strings.

### ETags vs Immutable — When to Use Which

> 💡 **Junior note — What are ETags?**
> An ETag is a "fingerprint" of a file. When the browser has a cached copy, it asks the server: "Has this file changed since fingerprint X?" If not, the server responds with `304 Not Modified` (no file transfer). If yes, it sends the new file.

> 💡 **Junior note — What is `immutable`?**
> `Cache-Control: immutable` tells the browser: "This file will NEVER change at this URL. Don't even bother asking." The browser skips the check entirely — zero network requests.

**They serve opposite purposes:**
- **ETags** = "check before reusing" (safe but costs a round-trip)
- **immutable** = "never check, just reuse" (fastest but stale if file changes)

**Rule of thumb:**
| Asset Type | Strategy | Why |
|-----------|----------|-----|
| CSS / JS / Fonts (versioned via `?ver=` or filename hash) | `immutable` + **disable ETags** | URL changes when content changes — safe to never revalidate |
| Images in `/uploads/` (no versioning) | ETags + normal `max-age` | URL stays the same even if image is re-uploaded — need revalidation |

Both server-specific guides implement this correctly.

---

> ⬅️ [Previous: Step 3 — Object Caching (Redis)](step-03-object-caching-redis.md) &nbsp;|&nbsp; ➡️ [Next: Step 5 — Minification & Concatenation](step-05-minification-concatenation.md)

