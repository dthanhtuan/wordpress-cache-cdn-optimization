# Task 1 — Step 4B: Browser Caching — Apache Configuration

> **When to use this guide:** Your server runs **Apache**. If you're on Nginx, use [Step 4A](step-04a-browser-caching-nginx.md) instead.
> **Prerequisites:** FTP/SSH access to WordPress root directory
> **Config file:** `/var/www/html/.htaccess` (in your WordPress root)
> **Read first:** [Step 4 — Browser Caching Overview](step-04-browser-caching.md) for shared concepts (cache-busting, ETags vs immutable)

---

## 📌 Why Apache?

Apache is the traditional web server for WordPress, commonly found on **shared hosting** (GoDaddy, Bluehost, SiteGround, cPanel servers). If you don't have SSH/root access and can only upload files via FTP, you're almost certainly on Apache.

**Apache advantages:**
- `.htaccess` files work **immediately** — no server reload needed. Save the file, refresh the page, done
- You can edit `.htaccess` via FTP, cPanel File Manager, or even a WordPress plugin — no SSH required
- `<IfModule>` wrappers make config safe — if a module isn't installed, Apache skips those rules instead of crashing

**Apache disadvantages compared to Nginx:**
- ⚠️ **Slower** — Apache re-reads `.htaccess` on **every single request** (Nginx loads config once into memory)
- ⚠️ **No FastCGI Cache** — Step 6 (server-level caching) uses Nginx FastCGI Cache. On Apache, you rely entirely on your page caching plugin (WP Rocket / W3TC) for page caching
- ⚠️ **No native Brotli** — most Apache installs don't have `mod_brotli`. You get Gzip only
- ⚠️ **Module dependency** — `mod_expires`, `mod_headers`, and `mod_deflate` must be enabled (usually are on shared hosting, but verify)

---

## 🔧 Detailed Steps

### 4B.1 — Check Required Apache Modules

Before configuring anything, verify these modules are enabled:

```bash
# If you have SSH access:
apache2ctl -M 2>/dev/null | grep -E "expires|headers|deflate"
# Or on CentOS/RHEL:
httpd -M 2>/dev/null | grep -E "expires|headers|deflate"

# Expected output (all three should appear):
#  expires_module (shared)
#  headers_module (shared)
#  deflate_module (shared)
```

If any are missing:
```bash
# Enable missing modules (Debian/Ubuntu):
sudo a2enmod expires
sudo a2enmod headers
sudo a2enmod deflate
sudo systemctl reload apache2
```

> 💡 **Junior note:** On shared hosting (cPanel), these modules are usually already enabled. If you can't run commands, just add the `.htaccess` rules — the `<IfModule>` wrappers will safely skip anything that's not available.

---

### 4B.2 — Add Browser Caching Headers

Edit `.htaccess` in your WordPress root directory (`/var/www/html/.htaccess` or wherever WordPress is installed).

> ⚠️ **IMPORTANT:** WordPress already has rules in `.htaccess` (the `# BEGIN WordPress` / `# END WordPress` block). Add your caching rules **ABOVE** the WordPress block. Never edit inside the WordPress block — WordPress overwrites it when you save permalink settings.

Add this at the **top** of `.htaccess`:

```apache
# ═══════════════════════════════════════════════════════════════════════════════
# Browser Caching — Static Assets
# Added by: Web Optimization Task 1, Step 4B
# ═══════════════════════════════════════════════════════════════════════════════

# ─── Expires Headers ──────────────────────────────────────────────────────────
# Tells the browser: "After downloading this file, keep it for X time."
# Uses MIME types (not file extensions) to identify file types.
<IfModule mod_expires.c>
    ExpiresActive On

    # Default — 1 hour for anything not explicitly listed
    ExpiresDefault                          "access plus 1 hour"

    # CSS & JavaScript — 1 year
    # Safe because WordPress uses cache-busting (?ver=1.2.0)
    # When content changes, the ?ver= changes → browser downloads fresh
    ExpiresByType text/css                  "access plus 1 year"
    ExpiresByType application/javascript    "access plus 1 year"
    ExpiresByType text/javascript           "access plus 1 year"

    # Images — 6 months
    # No cache-busting for uploads → need eventual revalidation
    ExpiresByType image/jpeg                "access plus 6 months"
    ExpiresByType image/png                 "access plus 6 months"
    ExpiresByType image/gif                 "access plus 6 months"
    ExpiresByType image/webp                "access plus 6 months"
    ExpiresByType image/avif                "access plus 6 months"
    ExpiresByType image/svg+xml             "access plus 6 months"
    ExpiresByType image/x-icon              "access plus 6 months"

    # Fonts — 1 year
    # Fonts are versioned with the theme — safe to cache long
    ExpiresByType font/woff                 "access plus 1 year"
    ExpiresByType font/woff2                "access plus 1 year"
    ExpiresByType application/font-woff     "access plus 1 year"
    ExpiresByType application/font-woff2    "access plus 1 year"
    ExpiresByType font/ttf                  "access plus 1 year"
    ExpiresByType application/x-font-ttf    "access plus 1 year"
    ExpiresByType font/eot                  "access plus 1 year"
    ExpiresByType font/otf                  "access plus 1 year"

    # HTML & XML — do NOT cache long
    # Page content changes frequently — let page caching plugin handle this
    ExpiresByType text/html                 "access plus 0 seconds"
    ExpiresByType application/xhtml+xml     "access plus 0 seconds"
</IfModule>

# ─── Cache-Control Headers ────────────────────────────────────────────────────
# Expires headers (above) and Cache-Control headers do similar things.
# Modern browsers prefer Cache-Control. We set both for maximum compatibility.
<IfModule mod_headers.c>
    # CSS & JS — immutable (browser should never revalidate)
    <FilesMatch "\.(css|js)$">
        Header set Cache-Control "public, max-age=31536000, immutable"
    </FilesMatch>

    # Images — standard caching (browser can revalidate via ETag)
    <FilesMatch "\.(jpg|jpeg|png|gif|webp|avif|svg|ico)$">
        Header set Cache-Control "public, max-age=15778476"
    </FilesMatch>

    # Fonts — immutable + CORS header for cross-origin loading
    <FilesMatch "\.(woff|woff2|ttf|eot|otf)$">
        Header set Cache-Control "public, max-age=31536000, immutable"
        Header set Access-Control-Allow-Origin "https://yourdomain.com"
        # ⚠️ Replace "https://yourdomain.com" with your actual domain.
        # If using a CDN, you may need:
        #   Header set Access-Control-Allow-Origin "https://cdn.yourdomain.com"
        # ⚠️ Do NOT use "*" — it lets anyone hotlink your fonts.
    </FilesMatch>
</IfModule>
```

> 💡 **Junior note — What is `<IfModule>`?**
> It means "only apply these rules if the module is installed." If `mod_expires` isn't enabled, Apache skips the entire block instead of throwing a 500 error. It's a safety net — your site won't break even if a module is missing (the caching just won't work until you enable it).

---

### 4B.3 — Configure ETag Headers

```apache
# ─── ETag Configuration ───────────────────────────────────────────────────────
# ETags = file fingerprints. Browser asks "has this changed?" → server checks ETag.
# For immutable files (CSS/JS/fonts), ETags are unnecessary — disable them.
# For images, keep ETags so browsers can efficiently revalidate after expiry.

<IfModule mod_headers.c>
    # Disable ETags for versioned/immutable assets
    # (already using immutable — ETag checks waste a round-trip)
    <FilesMatch "\.(css|js|woff|woff2|ttf|eot|otf)$">
        Header unset ETag
    </FilesMatch>
</IfModule>

# Keep ETags for images (non-immutable — need revalidation)
<FilesMatch "\.(jpg|jpeg|png|gif|webp|avif|svg|ico)$">
    FileETag MTime Size
</FilesMatch>

# Disable ETags for immutable assets at the FileETag level too
<FilesMatch "\.(css|js|woff|woff2|ttf|eot|otf)$">
    FileETag None
</FilesMatch>
```

---

### 4B.4 — Enable Gzip Compression

```apache
# ═══════════════════════════════════════════════════════════════════════════════
# Gzip Compression
# Compresses text-based files before sending to browser. Typical savings: 60-80%.
# ═══════════════════════════════════════════════════════════════════════════════
<IfModule mod_deflate.c>
    # Text-based formats (compresses well)
    AddOutputFilterByType DEFLATE text/plain
    AddOutputFilterByType DEFLATE text/html
    AddOutputFilterByType DEFLATE text/css
    AddOutputFilterByType DEFLATE text/javascript
    AddOutputFilterByType DEFLATE application/javascript
    AddOutputFilterByType DEFLATE application/json
    AddOutputFilterByType DEFLATE application/xml
    AddOutputFilterByType DEFLATE application/rss+xml
    AddOutputFilterByType DEFLATE image/svg+xml

    # ⚠️ Do NOT add font/woff or font/woff2!
    # WOFF uses zlib compression internally. WOFF2 uses Brotli internally.
    # Gzipping them wastes CPU with 0% size reduction and can even make
    # files slightly LARGER (gzip wrapper adds bytes to compressed data).

    # Don't compress tiny files (overhead > savings)
    DeflateCompressionLevel 6

    # Handle broken Accept-Encoding headers from old proxies
    BrowserMatch ^Mozilla/4 gzip-only-text/html
    BrowserMatch ^Mozilla/4\.0[678] no-gzip
    BrowserMatch \bMSIE !no-gzip !gzip-only-text/html
</IfModule>
```

> 💡 **Junior note — Where is Brotli?**
> Most Apache installations don't have `mod_brotli` installed. Unlike Nginx (where Brotli is a common add-on), Apache's Brotli support is rare on shared hosting. **Gzip is perfectly fine** — it gives you 60-80% compression. Brotli would add another 15-25% on top, but Gzip alone is a massive improvement over no compression.

---

### 4B.5 — Verify Changes (No Reload Needed!)

Unlike Nginx, Apache reads `.htaccess` on every request. **Save the file and it works immediately.**

```bash
# Check CSS headers
curl -I https://yourdomain.com/wp-content/themes/mytheme/style.css
# Look for:
#   Cache-Control: public, max-age=31536000, immutable
#   (No ETag header)

# Check image headers
curl -I https://yourdomain.com/wp-content/uploads/2024/01/photo.jpg
# Look for:
#   Cache-Control: public, max-age=15778476
#   ETag: "..."  (should be present)

# Check font headers
curl -I https://yourdomain.com/wp-content/themes/mytheme/fonts/font.woff2
# Look for:
#   Cache-Control: public, max-age=31536000, immutable
#   Access-Control-Allow-Origin: https://yourdomain.com
#   (No ETag header)

# Check Gzip
curl -I -H "Accept-Encoding: gzip" https://yourdomain.com/
# Look for:
#   Content-Encoding: gzip
```

---

## ⚠️ Apache-Specific Risks

### Risk 1: `.htaccess` Syntax Error = 500 Error

A typo in `.htaccess` causes an **immediate 500 Internal Server Error** on your entire site. Unlike Nginx (which validates before applying), Apache applies `.htaccess` changes instantly — including broken ones.

**Prevention:**
```bash
# ALWAYS keep a backup before editing:
cp /var/www/html/.htaccess /var/www/html/.htaccess.backup

# If you get a 500 error, restore immediately:
cp /var/www/html/.htaccess.backup /var/www/html/.htaccess
```

> 💡 **Junior note:** If you can't SSH in, use FTP/cPanel File Manager to rename `.htaccess` to `.htaccess.broken` and rename `.htaccess.backup` to `.htaccess`. Site comes back instantly.

### Risk 2: WordPress Overwrites `.htaccess`

WordPress rewrites the `# BEGIN WordPress` / `# END WordPress` block when you save permalink settings. If you put your caching rules INSIDE that block, they'll be deleted.

**Prevention:** Always add custom rules **ABOVE** the `# BEGIN WordPress` line.

### Risk 3: Hosting Provider Overrides

Some shared hosting providers add their own caching rules or override `.htaccess` directives at the server level. If your rules don't seem to work:
1. Check with your hosting provider
2. Look for a caching section in cPanel / Plesk
3. Some hosts (SiteGround, WP Engine) have their own caching that supersedes `.htaccess`

---

## ✅ Validation Checklist

| # | Check | Command | Expected |
|---|-------|---------|----------|
| 1 | `.htaccess` backup exists | `ls -la .htaccess.backup` | File exists |
| 2 | No 500 error | Visit your site in browser | Site loads normally |
| 3 | CSS has immutable header | `curl -I ...style.css` | `Cache-Control: public, max-age=31536000, immutable` |
| 4 | JS has immutable header | `curl -I ...app.js` | `Cache-Control: public, max-age=31536000, immutable` |
| 5 | Image has 6-month header | `curl -I ...photo.jpg` | `Cache-Control: public, max-age=15778476` |
| 6 | Font has CORS header | `curl -I ...font.woff2` | `Access-Control-Allow-Origin: https://yourdomain.com` |
| 7 | No ETag on CSS/JS/fonts | `curl -I ...style.css` | ETag header **absent** |
| 8 | ETag on images | `curl -I ...photo.jpg` | ETag header **present** |
| 9 | Gzip working | `curl -I -H "Accept-Encoding: gzip" https://yourdomain.com/` | `Content-Encoding: gzip` |
| 10 | Cache-busting works | Deploy CSS change → hard refresh | New CSS loads |
| 11 | Custom rules above WP block | `head -20 .htaccess` | Your rules appear before `# BEGIN WordPress` |
| 12 | PageSpeed score improved | Run PageSpeed Insights | "Leverage browser caching" warning gone |

---

## 🚨 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| 500 Internal Server Error | Syntax error in `.htaccess` | Restore from backup: `cp .htaccess.backup .htaccess` |
| Headers not appearing | `mod_expires` or `mod_headers` not enabled | Run `sudo a2enmod expires headers && sudo systemctl reload apache2` |
| Gzip not working | `mod_deflate` not enabled | Run `sudo a2enmod deflate && sudo systemctl reload apache2` |
| Fonts CORS error in browser console | `Access-Control-Allow-Origin` missing or wrong domain | Check FilesMatch block — domain must match exactly |
| Rules disappeared after saving permalinks | Rules were inside `# BEGIN WordPress` block | Move rules ABOVE the WordPress block, restore from backup |
| Headers show different values | Hosting provider overrides | Check cPanel/Plesk caching settings, contact host support |
| `<IfModule>` rules being skipped | Module not installed | Check with `apache2ctl -M` — enable missing modules |

---

## 📝 Note About Step 6 (Server-Level Caching)

Step 6 configures **Nginx FastCGI Cache**, which is Nginx-specific. If you're on Apache, you **cannot use FastCGI Cache**. Instead, you rely entirely on your page caching plugin:

- **WP Rocket** (Step 2A) → generates static HTML files that Apache serves efficiently
- **W3 Total Cache** with "Disk: Enhanced" (Step 2B) → also generates static `.html` files

This means your caching stack on Apache is:
```
Apache Stack:
Browser Cache (.htaccess) → Page Cache (plugin) → Object Cache (Redis) → MySQL

vs Nginx Stack:
Browser Cache (nginx.conf) → FastCGI Cache (Nginx) → Page Cache (plugin) → Object Cache (Redis) → MySQL
```

Apache has one fewer caching layer, but the page caching plugin compensates well for most sites.

---

> ⬅️ [Back to Step 4 — Browser Caching Overview](step-04-browser-caching.md) &nbsp;|&nbsp; ➡️ [Next: Step 5 — Minification & Concatenation](step-05-minification-concatenation.md)
