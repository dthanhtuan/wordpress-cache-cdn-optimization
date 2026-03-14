# Task 2 — Step 4: Install & Configure WordPress CDN Plugin

> **Estimated Duration:** 1 day
> **Goal:** Configure WordPress to rewrite media URLs so browsers load images from the CDN instead of your origin server.
> **Risk Level:** 🟡 Low — easily reversible by deactivating the plugin.

---

## 🧠 How WordPress Connects to the CDN (Read This First)

> 💡 **Junior note:** After Step 2, your CDN is set up and waiting. But your WordPress site still has no idea the CDN exists. Every image URL in your HTML still points to `yourdomain.com`. The CDN is like a warehouse you built but nobody delivers to — you need to change the delivery address.

### The Problem

Right now, your WordPress pages contain image URLs like this:
```html
<img src="https://yourdomain.com/wp-content/uploads/2024/01/photo.jpg">
```

The browser sees `yourdomain.com` and downloads the image directly from your server. Your CDN (`cdn.yourdomain.com`) exists but nobody uses it.

### The Solution: URL Rewriting

A CDN plugin intercepts WordPress's HTML output **before it's sent to the browser** and replaces the origin domain with the CDN domain:

```
What WordPress generates:
<img src="https://yourdomain.com/wp-content/uploads/2024/01/photo.jpg">

What the CDN plugin changes it to (in the HTML, before it reaches the browser):
<img src="https://cdn.yourdomain.com/wp-content/uploads/2024/01/photo.jpg">
```

Now the browser sees `cdn.yourdomain.com` and downloads the image from the CDN edge node instead of your server.

### What The Plugin Does NOT Do

> ⚠️ **Common misconception:** The CDN plugin does NOT upload/push/copy images to the CDN. With a Pull Zone CDN (BunnyCDN), your images stay on your server. The CDN fetches them automatically when requested. The plugin's only job is **rewriting URLs in the HTML output**.

```
What actually happens:

1. Visitor requests yourdomain.com/about/
2. WordPress generates the HTML page (with images pointing to yourdomain.com)
3. CDN plugin intercepts the HTML and rewrites image URLs to cdn.yourdomain.com
4. Browser receives HTML with CDN URLs
5. Browser requests image from cdn.yourdomain.com
6. CDN checks: "Do I have this image cached?"
   → YES: serves from edge cache (fast)
   → NO: fetches from yourdomain.com, caches it, then serves it
7. Your server still has all the original images — nothing moved
```

### Two Layers of URL Rewriting

The CDN plugin only rewrites URLs **on-the-fly** in the HTML output. This handles:
- ✅ All images displayed on pages/posts
- ✅ CSS/JS file URLs
- ✅ Theme images, plugin assets

But it does **NOT** change URLs stored in the database. Existing content in `wp_posts.post_content` still contains `yourdomain.com/wp-content/uploads/...` — the plugin rewrites them at render time. This is usually fine, but Step 6 (migration script) does a permanent database rewrite for thoroughness.

---

## 📌 Which Plugin Should You Use?

It depends on what caching plugin you chose in Task 1 and what CDN type you're using:

| Your Setup | Recommended Plugin | Why |
|-----------|-------------------|-----|
| **WP Rocket + Pull Zone CDN** (BunnyCDN) | WP Rocket's built-in CDN settings | No extra plugin needed — it's already installed |
| **W3 Total Cache + Pull Zone CDN** (BunnyCDN) | W3TC's built-in CDN settings | No extra plugin needed — it's already installed |
| **No caching plugin + Pull Zone CDN** | CDN Enabler (free) | Lightweight, does one thing well |
| **Push to storage** (S3, BunnyCDN Storage Zone) | WP Offload Media ($99/year) | Actually copies files to external storage |

> 💡 **Junior note — Pull Zone vs Push:**
> - **Pull Zone** (what we use): Images stay on your server. CDN fetches them on demand. Plugin just rewrites URLs.
> - **Push / Storage Zone**: Images are actually copied/uploaded to CDN storage. Your server may not keep copies. This is more complex and usually not needed.

### Full Plugin Comparison

| Feature | WP Rocket CDN | W3TC CDN | CDN Enabler | WP Offload Media |
|---------|:------------:|:--------:|:-----------:|:----------------:|
| Price | Included with WP Rocket | Free (included) | Free | $99/year |
| Setup complexity | 🟢 30 seconds | 🟡 5 minutes | 🟢 1 minute | 🟠 15 minutes |
| URL rewriting | ✅ | ✅ | ✅ | ✅ |
| Choose which file types | ✅ | ✅ | ✅ | ✅ |
| Exclude specific paths | ✅ | ✅ | ✅ | ✅ |
| Works with any Pull Zone CDN | ✅ | ✅ | ✅ | ✅ |
| Copies files to external storage | ❌ | ❌ | ❌ | ✅ |
| Removes files from origin | ❌ | ❌ | ❌ | ✅ (optional) |

---

## 🔧 Detailed Steps

### Option A — WP Rocket Built-In CDN (If You Use WP Rocket)

> ✅ **Recommended if WP Rocket is already installed from Task 1.** No additional plugin needed.

Navigate to **Settings → WP Rocket → CDN** tab:

```
✅ Enable Content Delivery Network

CDN CNAME(s):    cdn.yourdomain.com
   → This is the CDN domain you set up in Step 2.
   → WP Rocket will rewrite all matching URLs to use this domain.

Included file types:  All files
   → Or specify: .jpg, .jpeg, .png, .gif, .webp, .avif, .svg, .ico, .css, .js, .woff, .woff2
   → "All files" is usually fine — WP Rocket already excludes PHP files.

Excluded files (leave default unless needed):
   → WP Rocket auto-excludes .php files and wp-admin assets
   → Add any custom exclusions if needed (e.g., files that must come from origin)
```

**What happens after saving:**
1. WP Rocket hooks into WordPress's `template_redirect` output buffer
2. Before sending HTML to the browser, it finds all URLs matching `yourdomain.com/wp-content/` and `yourdomain.com/wp-includes/`
3. It replaces `yourdomain.com` with `cdn.yourdomain.com` in those URLs
4. Browser receives the rewritten HTML and loads assets from CDN

**Verify immediately:**
```bash
# View page source and check for CDN URLs
curl -s https://staging.yourdomain.com/ | grep -o 'https://cdn\.yourdomain\.com[^"]*' | head -5
# Should show CDN URLs for images, CSS, JS

# Verify NO remaining origin URLs for static assets
curl -s https://staging.yourdomain.com/ | grep -o 'https://yourdomain\.com/wp-content/uploads/[^"]*' | head -5
# Should return NOTHING (all rewritten to CDN)
```

---

### Option B — W3 Total Cache Built-In CDN (If You Use W3TC)

Navigate to **Performance → General Settings**:
```
CDN:                    ✅ Enable
CDN Type:               Generic Mirror (this is Pull Zone)
```

Navigate to **Performance → CDN**:
```
Configuration:
   Site's hostname:              yourdomain.com
   Replace site's hostname with: cdn.yourdomain.com

Included file types:
   ✅ Host attachments (wp-content/uploads/)
   ✅ Host wp-includes/ files
   ✅ Host theme files
   ✅ Host custom files (if needed)

Advanced:
   Rejected files:  (leave empty unless you need to exclude specific files)
   Rejected user agents:  (leave empty)
```

**Verify:**
```bash
curl -s https://staging.yourdomain.com/ | grep -o 'https://cdn\.yourdomain\.com[^"]*' | head -5
```

---

### Option C — CDN Enabler Plugin (If You Don't Use WP Rocket or W3TC)

> Use this lightweight standalone plugin if you're not running WP Rocket or W3TC.

```bash
wp plugin install cdn-enabler --activate
```

Navigate to **Settings → CDN Enabler**:

```
CDN Hostname:    cdn.yourdomain.com
   → The CDN domain from Step 2.

Included Directories:   wp-content,wp-includes
   → Which directories to rewrite.
   → wp-content = uploads, themes, plugins
   → wp-includes = WordPress core CSS/JS

Exclusions:      .php
   → NEVER serve PHP files from CDN — they need to execute on your server.
   → Add other exclusions if needed: .xml (sitemaps should come from origin)

✅ Enable HTTPS
   → Ensures all rewritten URLs use https://cdn.yourdomain.com

⚠️ Enable relative paths
   → Converts absolute URLs to protocol-relative (//cdn.yourdomain.com/...)
   → Generally safe, but can cause issues on some page builders.
   → If images break after enabling, disable this option.
```

**How CDN Enabler works under the hood:**
```php
// CDN Enabler registers an output buffer callback:
ob_start( 'cdn_enabler_rewrite' );

// Before the HTML is sent to the browser, it runs:
// str_replace( 'yourdomain.com/wp-content/', 'cdn.yourdomain.com/wp-content/', $html )
// str_replace( 'yourdomain.com/wp-includes/', 'cdn.yourdomain.com/wp-includes/', $html )

// That's it — simple string replacement in the HTML output.
```

---

### Option D — WP Offload Media (Push to Storage — Advanced)

> Only use this if you want to **actually copy files** to external storage (BunnyCDN Storage Zone, AWS S3, DigitalOcean Spaces). For Pull Zone setups, use Options A/B/C instead.

```bash
wp plugin install amazon-s3-and-cloudfront --activate
```

Navigate to **Settings → Offload Media**:
```
Storage Provider:       BunnyCDN / Amazon S3 / DigitalOcean Spaces
Bucket/Zone:            your-cdn-storage-zone
Access Key:             (from your CDN/S3 dashboard)
Secret Key:             (from your CDN/S3 dashboard)

✅ Copy Files to Bucket     → uploads images to external storage when added to Media Library
⚠️ Remove Files from Server → deletes local copy after uploading (saves disk space but risky)
   → Recommend: leave OFF initially. Keep files on both origin and CDN.
   → Only enable after confirming everything works and you have backups.

Delivery:
   CDN URL:             https://cdn.yourdomain.com
   Path:                /wp-content/uploads/
```

> ⚠️ **WP Offload Media is fundamentally different** from Options A/B/C. Those just rewrite URLs (files stay on your server). This actually moves files to external storage. It's more complex, harder to rollback, and costs $99/year. Only use it if you need to reduce origin disk usage.

---

## 3.1 — What Gets Rewritten (And What Doesn't)

Understanding exactly which URLs get rewritten helps you debug issues:

| URL Type | Rewritten to CDN? | Example |
|---------|:-----------------:|---------|
| Images in post content | ✅ Yes | `<img src="cdn.yourdomain.com/wp-content/uploads/photo.jpg">` |
| Theme CSS/JS | ✅ Yes | `<link href="cdn.yourdomain.com/wp-content/themes/mytheme/style.css">` |
| Plugin CSS/JS | ✅ Yes | `<script src="cdn.yourdomain.com/wp-content/plugins/woo/assets/js/...">` |
| WordPress core JS | ✅ Yes | `cdn.yourdomain.com/wp-includes/js/jquery/jquery.min.js` |
| Images in CSS (background-image) | ✅ Yes (most plugins) | `background: url(cdn.yourdomain.com/wp-content/themes/...)` |
| **Admin dashboard assets** | ❌ No | wp-admin should always load from origin |
| **PHP files** | ❌ No | `wp-login.php`, `admin-ajax.php` must run on your server |
| **REST API** | ❌ No | `/wp-json/` endpoints must hit your server |
| **Sitemaps** | ❌ No | `/sitemap.xml` should be served by your server |
| **URLs in database** | ❌ No | `wp_posts.post_content` still stores origin URLs — rewritten at render time only |
| **Inline `<img>` in Gutenberg blocks** | ✅ Yes | Rewritten in HTML output buffer |
| **Shortcode-generated images** | ✅ Yes | As long as the shortcode outputs standard `<img>` tags |

---

## 3.2 — Configure WebP Delivery

WebP is a modern image format that's **25-35% smaller** than JPEG at the same quality. All modern browsers support it. There are two ways to serve WebP:

### Method 1: BunnyCDN Optimizer (Recommended — No WordPress Config Needed)

BunnyCDN can automatically convert and serve WebP at the edge — your server doesn't need to do anything:

In BunnyCDN Pull Zone → **Optimizer** tab:
```
✅ Enable Bunny Optimizer
✅ WebP Conversion
   → BunnyCDN automatically converts JPEG/PNG to WebP when the browser supports it
   → Original file stays as-is on your server
   → CDN detects the browser's Accept header and serves the right format
✅ Lazy Loading (optional — enable if not already done by WP Rocket/theme)
```

**How it works:**
```
Browser sends:    Accept: image/webp,image/apng,image/*
CDN receives:     Request for photo.jpg
CDN responds:     Serves photo.jpg converted to WebP format (smaller file)
                  Content-Type: image/webp

Browser sends:    Accept: image/jpeg,image/*  (old Safari, no WebP)
CDN responds:     Serves original photo.jpg as JPEG
                  Content-Type: image/jpeg
```

### Method 2: WordPress-Side WebP (Using Imagify or ShortPixel)

If your CDN doesn't have an optimizer, generate WebP files on your server:

```bash
wp plugin install imagify --activate
```

**Settings → Imagify:**
```
Optimization level:  Normal (recommended) or Aggressive
✅ Convert images to WebP
✅ Display images in WebP format on the frontend
   Method: Use <picture> tags
   → Generates both photo.jpg and photo.jpg.webp on your server
   → Outputs <picture> tags so browsers pick the right format:

   <picture>
     <source srcset="cdn.yourdomain.com/.../photo.jpg.webp" type="image/webp">
     <img src="cdn.yourdomain.com/.../photo.jpg" type="image/jpeg">
   </picture>
```

> 💡 **Junior note:** Method 1 (BunnyCDN Optimizer) is easier — zero WordPress config, automatic conversion at the CDN edge. Method 2 gives you more control but requires a WordPress plugin and generates extra files on your server.

---

## 3.3 — Test CDN URL Rewriting on Staging

After configuring your chosen plugin, verify everything is rewritten correctly:

```bash
# ── Test 1: Check image URLs in page source ──────────────────────────────────
curl -s https://staging.yourdomain.com/ | grep -oP 'src="[^"]*\.(jpg|jpeg|png|gif|webp|svg)[^"]*"' | head -10
# ALL image URLs should start with https://cdn.yourdomain.com/
# NONE should start with https://yourdomain.com/ or https://staging.yourdomain.com/

# ── Test 2: Check CSS URLs ───────────────────────────────────────────────────
curl -s https://staging.yourdomain.com/ | grep -oP 'href="[^"]*\.css[^"]*"' | head -5
# Should show cdn.yourdomain.com/wp-content/themes/... URLs

# ── Test 3: Check JS URLs ────────────────────────────────────────────────────
curl -s https://staging.yourdomain.com/ | grep -oP 'src="[^"]*\.js[^"]*"' | head -5
# Should show cdn.yourdomain.com/wp-content/... and cdn.yourdomain.com/wp-includes/...

# ── Test 4: Check that admin URLs are NOT rewritten ──────────────────────────
# Log in to wp-admin and check page source — admin assets should NOT use CDN

# ── Test 5: Check multiple pages (not just home) ─────────────────────────────
curl -s https://staging.yourdomain.com/about/ | grep -c 'cdn\.yourdomain\.com'
curl -s https://staging.yourdomain.com/blog/ | grep -c 'cdn\.yourdomain\.com'
# Should return a count > 0 (CDN URLs found on each page)

# ── Test 6: Images actually load from CDN ─────────────────────────────────────
# Pick any CDN image URL from above and fetch it directly:
curl -I https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# HTTP/2 200 + x-cache: HIT (or MISS on first request)
```

---

## 3.4 — Test New Uploads Route Through CDN

```
1. Go to WordPress Admin → Media → Add New
2. Upload a test image (any .jpg or .png)
3. Go to Media Library → click the uploaded image → "Copy URL to clipboard"
4. Check the URL:

   ✅ Correct: https://cdn.yourdomain.com/wp-content/uploads/2026/03/test-image.jpg
   ❌ Wrong:   https://yourdomain.com/wp-content/uploads/2026/03/test-image.jpg

5. Insert the image into a post → save → view the post
6. View page source → find the <img> tag → URL should be cdn.yourdomain.com
```

> 💡 **Junior note — Why does the Media Library URL matter?**
> When you insert an image into a post, WordPress stores the URL in the database. If the URL shows `yourdomain.com` in the Media Library but the CDN plugin rewrites it to `cdn.yourdomain.com` in the page HTML, the plugin is working correctly. The database stores the origin URL; the plugin rewrites on-the-fly. Both behaviors are normal.
>
> For Pull Zone CDN, the origin URL in the database is actually fine — the CDN plugin handles rewriting. Step 6's migration script is an optional extra step for consistency.

---

## 3.5 — What About Existing Images Already in the Database?

After enabling the CDN plugin:
- **New pages/posts you view:** Images are rewritten to CDN URLs ✅
- **Existing images in post_content (database):** Still stored as `yourdomain.com` in the database, but rewritten at render time by the plugin ✅
- **RSS feeds, REST API, email:** May still show origin URLs (depends on plugin) ⚠️
- **Custom database tables from plugins:** Not rewritten (plugin only touches HTML output) ⚠️

**Step 6** (media migration script) does a permanent database-level rewrite for these edge cases. For most sites, the on-the-fly rewriting from this step is sufficient.

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | CDN plugin installed and active | `wp plugin list --status=active` shows plugin |
| 2 | CDN URL configured correctly | Plugin settings show `cdn.yourdomain.com` |
| 3 | Home page: images use CDN URLs | `curl -s https://staging.yourdomain.com/ \| grep cdn.yourdomain.com` |
| 4 | Inner page: images use CDN URLs | Check 2-3 blog posts and pages |
| 5 | CSS files use CDN URLs | View source → `<link>` tags show `cdn.yourdomain.com` |
| 6 | JS files use CDN URLs | View source → `<script>` tags show `cdn.yourdomain.com` |
| 7 | No origin URLs remain for static assets | `curl -s ... \| grep 'yourdomain.com/wp-content/uploads/'` returns nothing |
| 8 | New uploads work with CDN | Upload image → insert in post → URL shows CDN domain in rendered page |
| 9 | Admin dashboard NOT rewritten | wp-admin assets load from origin (not CDN) |
| 10 | WebP served to Chrome | DevTools → Network → filter images → `Content-Type: image/webp` |
| 11 | JPEG/PNG fallback for old browsers | Test with Safari or user-agent that doesn't support WebP |
| 12 | No broken images on any page | Browse 10+ staging pages → all images visible |
| 13 | HTTPS on all CDN URLs | All rewritten URLs start with `https://` |
| 14 | No mixed content warnings | Browser console → no "Mixed Content" errors |
| 15 | PHP files NOT rewritten | `wp-login.php`, `admin-ajax.php` still on origin domain |

---

> ⬅️ [Previous: Step 3 — Image Optimization Pipeline](step-03-image-optimization-pipeline.md) &nbsp;|&nbsp; ➡️ [Next: Step 5 — CDN Caching & Security](step-05-cdn-caching-security.md)

