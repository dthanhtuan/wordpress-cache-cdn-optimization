# Task 2 — Step 3: Configure Image Optimization Pipeline

> **Estimated Duration:** 1.5 days
> **Goal:** Ensure all images — existing and new — are automatically compressed, resized, converted to WebP, lazy-loaded, and served responsively.
> **Risk Level:** 🟡 Low — most features are built into WordPress core. Only compression needs a plugin or code.

---

## 📋 Project Requirement (Section 3.2.3)

> **3.2.3. Image optimization**
> Configure automatic image optimization upon upload:
> - Lossless compression
> - Resize to a maximum width of 1920px
> - Automatic preview generation
> - Implement lazy loading for images
> - Configure adaptive images for different devices

### What WordPress Already Handles for Free

Before installing anything, understand that **4 out of 5 requirements are already built into WordPress core**:

| Requirement | WordPress Core? | Since Version | What You Need to Do |
|---|---|---|---|
| **Resize to max 1920px** | ✅ Built-in | WP 5.3+ | Just verify the `big_image_size_threshold` filter (default = 2560px, we change to 1920px) |
| **Automatic preview generation** | ✅ Built-in | Always | WordPress generates thumbnail, medium, large on every upload |
| **Lazy loading** | ✅ Built-in | WP 5.5+ | Adds `loading="lazy"` automatically to all `<img>` tags |
| **Adaptive images (srcset)** | ✅ Built-in | WP 4.4+ | Generates `srcset` attribute automatically — browser picks best size |
| **Lossless compression** | ❌ NOT built-in | — | Need plugin (EWWW) or custom code |

> **Key takeaway:** Don't install 5 plugins for 5 features. WordPress already does most of the work. We only need to add **compression + WebP conversion**.

---

## 🎯 Objectives
- ✅ Verify WordPress built-in features are working (resize, thumbnails, lazy load, srcset)
- 🔧 Configure max resize to 1920px (change from default 2560px)
- 🔧 Install EWWW Image Optimizer for lossless compression + WebP
- 🔧 Bulk optimize existing images in the media library
- 🔧 Remove unnecessary image sizes to reduce upload bloat

---

## 🔧 Part 1: WordPress Built-in Features (No Plugin Needed)

### 3.1 — Configure Max Image Resize (1920px)

WordPress 5.3+ automatically scales down uploaded images that are larger than the `big_image_size_threshold` (default: 2560px). We change this to 1920px per the requirement.

Add to your **active theme's `functions.php`** or a custom plugin:

```php
/**
 * Resize uploaded images to max 1920px width/height
 * 
 * How it works:
 * - User uploads a 4000x3000px photo
 * - WordPress automatically scales it to 1920x1440px
 * - The original (4000px) is saved as image-scaled.jpg (backup)
 * - All subsequent sizes (thumbnail, medium, large) are generated from the 1920px version
 *
 * Why 1920px:
 * - Matches Full HD resolution (1920x1080)
 * - Larger images waste bandwidth — no monitor can display more without zooming
 * - Requirement 3.2.3 specifies "maximum width of 1920px"
 */
add_filter('big_image_size_threshold', function() {
    return 1920;
});
```

> **What about existing images larger than 1920px?** They won't be resized retroactively. We'll handle those with EWWW's bulk optimizer in Part 2, or with `wp media regenerate` in section 3.3.

---

### 3.2 — Configure WordPress Image Sizes

WordPress generates multiple sizes for every uploaded image. Customize which sizes are created:

```php
/**
 * Customize WordPress image sizes
 * Add to functions.php
 */

// ── Remove sizes you don't use (reduces disk space + upload time) ──
add_filter('intermediate_image_sizes_advanced', function($sizes) {
    // WordPress default sizes you might not need:
    unset($sizes['medium_large']); // 768px — rarely used in themes
    // Keep: thumbnail (150x150), medium (300x300), large (1024x1024)
    return $sizes;
});

// ── Add custom sizes your theme actually uses ──
add_image_size('hero',  1920, 9999, false); // Full-width hero (no crop)
add_image_size('card',   640,  480, true);  // Card thumbnails (cropped to exact ratio)

// ── Set default sizes to sensible values ──
// Settings → Media in admin, or via code:
update_option('large_size_w', 1200);
update_option('large_size_h', 9999);   // 9999 = no height limit (preserve aspect ratio)
update_option('medium_size_w', 600);
update_option('medium_size_h', 9999);
update_option('thumbnail_size_w', 300);
update_option('thumbnail_size_h', 300);
update_option('thumbnail_crop', 1);     // Crop thumbnails to exact square
```

> **How many sizes does WordPress generate per upload?**
> Default: 4 sizes (thumbnail, medium, medium_large, large) + original = **5 files per image**.
> With our config: 5 sizes (thumbnail, medium, large, hero, card) + original = **6 files per image**.
> Upload 1 photo → WordPress creates 6 files. With WebP → **12 files**. This is why removing unused sizes matters.

---

### 3.3 — Regenerate Thumbnails for Existing Images

After changing image sizes, existing images still have the OLD sizes. Regenerate them:

```bash
# Count total images to regenerate
wp post list --post_type=attachment --post_mime_type=image --format=count

# Regenerate all thumbnails
# ⚠️ This is CPU-intensive — run during LOW TRAFFIC hours
wp media regenerate --yes

# For large libraries (5,000+ images), run in batches to avoid timeout:
wp media regenerate --yes --batch-size=100

# If you only want to generate MISSING sizes (doesn't recreate existing):
wp media regenerate --only-missing --yes
```

> **⚠️ Risk:** On a site with 10,000 images × 6 sizes = regenerating 60,000 files. This can take 1-2 hours and will spike CPU to 100%. Always run on staging first, then on production during maintenance windows.

---

### 3.4 — Verify Lazy Loading (Built-in)

WordPress 5.5+ automatically adds `loading="lazy"` to images. You don't need a plugin for this.

```php
// WordPress automatically does this for:
// ✅ the_post_thumbnail()
// ✅ wp_get_attachment_image()
// ✅ Images in post content (the_content filter)

// The FIRST image on the page skips lazy loading (LCP optimization, WP 5.9+)
// This is correct behavior — the hero image should load immediately

// ── Verify in your theme ──
// Your theme MUST use WordPress image functions:
// ✅ CORRECT — WordPress adds loading="lazy" + srcset automatically
the_post_thumbnail('large');
echo wp_get_attachment_image($attachment_id, 'large');

// ❌ WRONG — bypasses lazy loading AND srcset
echo '<img src="' . get_the_post_thumbnail_url() . '">';
// Fix: Use wp_get_attachment_image() instead
```

**How to verify lazy loading is working:**
1. Open your site in Chrome
2. DevTools → Elements tab → search for `loading="lazy"`
3. You should see it on all `<img>` tags EXCEPT the first image

> **Do you need a lazy loading plugin (like a3 Lazy Load)?** Only if:
> - Your theme uses raw `<img>` tags instead of WordPress functions (bad theme)
> - You need to lazy-load background images (`background-image: url(...)`)
> - You need to lazy-load iframes/videos too
>
> For most sites, WordPress built-in lazy loading is sufficient.

---

### 3.5 — Verify Responsive Images / srcset (Built-in)

WordPress 4.4+ automatically generates `srcset` attribute on images. The browser picks the best size for the user's screen.

```html
<!-- What WordPress automatically generates: -->
<img src="photo-1200x800.jpg"
     srcset="photo-300x200.jpg 300w,
             photo-600x400.jpg 600w,
             photo-1200x800.jpg 1200w,
             photo-1920x1280.jpg 1920w"
     sizes="(max-width: 1920px) 100vw, 1920px"
     loading="lazy"
     alt="...">

<!-- What the browser does:
     - Phone (375px wide):   downloads photo-600x400.jpg    (smallest that fits)
     - Tablet (768px wide):  downloads photo-1200x800.jpg
     - Desktop (1920px):     downloads photo-1920x1280.jpg
     → Each device downloads ONLY the size it needs = bandwidth savings -->
```

**Customize the `sizes` attribute for specific layouts:**

```php
/**
 * Tell the browser how wide the image will be at different viewport sizes.
 * Default: "(max-width: {image_width}px) 100vw, {image_width}px"
 * This assumes full-width images — override if your layout is different.
 */
add_filter('wp_calculate_image_sizes', function($sizes, $size, $image_src, $image_meta, $attachment_id) {
    // Full-width hero images
    if ($size === 'hero') {
        return '100vw';
    }
    // Images inside a content column (e.g., 800px max-width article)
    if ($size === 'large' || $size === 'medium') {
        return '(max-width: 800px) 100vw, 800px';
    }
    return $sizes;
}, 10, 5);
```

---

## 🔧 Part 2: Image Compression + WebP (EWWW Image Optimizer)

### Why EWWW Over Imagify / ShortPixel / Smush?

The popular image optimization plugins have a hidden problem — they're **not really free** for production:

| Plugin | Free Tier Limit | Reality for Active Site |
|---|---|---|
| **Imagify** | 25 images/month | 1 blog post with 5 images × 6 sizes = 30 images. **Exceeded in 1 post.** |
| **ShortPixel** | 100 images/month | 3-4 blog posts and you're done for the month. |
| **Smush Free** | Bulk optimize, no WebP | No WebP conversion, 5MB max per image, no lossy compression. |
| **EWWW Free (Local)** | **Unlimited** | Runs compression tools on YOUR server — no API, no limits, no cost. |

### How EWWW Free Mode Works

```
Cloud plugins (Imagify, ShortPixel):
  Upload image → Send to cloud API → API compresses → Returns to you
  ⚠️ Monthly limit, API key required, images leave your server

EWWW Local/Free mode:
  Upload image → Server tools (jpegoptim, optipng, cwebp) compress locally
  ✅ Unlimited, no API key, images never leave your server
  ⚠️ Uses YOUR server's CPU (fine for most sites)
```

---

### 3.6 — Install Server-Side Compression Tools

EWWW's free mode needs these tools installed on your server:

```bash
# Ubuntu/Debian:
sudo apt-get update
sudo apt-get install -y jpegoptim optipng webp

# CentOS/RHEL:
sudo yum install -y jpegoptim optipng libwebp-tools

# Verify installation:
jpegoptim --version    # JPEG lossless compression
optipng --version      # PNG lossless optimization
cwebp -version         # WebP conversion tool

# Check paths (EWWW needs to find these):
which jpegoptim        # /usr/bin/jpegoptim
which optipng          # /usr/bin/optipng
which cwebp            # /usr/bin/cwebp
```

> **⚠️ Shared hosting?** Most shared hosts (SiteGround, Bluehost) don't let you install system packages. In that case, EWWW bundles its own binaries — but performance is worse. If on shared hosting, consider EWWW's paid API mode ($0.003/image) or ShortPixel.

---

### 3.7 — Install & Configure EWWW Image Optimizer

```bash
# Install EWWW Image Optimizer (free from WordPress.org)
wp plugin install ewww-image-optimizer --activate

# For Multisite — network activate:
wp plugin install ewww-image-optimizer --activate-network
```

**Settings → EWWW Image Optimizer:**

```
──── Optimize Tab ────
Compression mode:       ✅ Pixel Perfect (Lossless)
                        ← Matches requirement: "Lossless compression"
                        ← No visible quality loss, typically 10-30% file size reduction

Remove Metadata:        ✅ Yes
                        ← Strips EXIF data (camera model, GPS location, etc.)
                        ← Saves 20-50KB per photo
                        ← ⚠️ Also removes copyright info — acceptable for most sites

──── Resize Tab ────
Resize Images:          ✅ Enable
Max Width:              1920
Max Height:             1920
                        ← Redundant with big_image_size_threshold, but acts as safety net

──── WebP Tab ────
WebP Conversion:        ✅ Enable
WebP delivery method:   Choose based on your server:
```

### WebP Delivery Methods

EWWW offers multiple ways to serve WebP to browsers that support it:

**Method 1: `<picture>` element (Recommended for Nginx)**
```
EWWW Setting: "JS WebP Rewriting" = ON

How it works:
- EWWW generates .webp version alongside every .jpg/.png
- JavaScript rewrites <img> to <picture> with WebP source
- Browser picks WebP if supported, falls back to original

Pros: Works on any server (Nginx, Apache, CDN)
Cons: Requires JavaScript (tiny overhead)
```

**Method 2: `.htaccess` rewrite (Apache only)**
```
EWWW Setting: "WebP Rewriting" = ON (Apache only)

How it works:
- EWWW adds rewrite rules to .htaccess
- Apache checks if browser sends Accept: image/webp header
- If yes → serves .webp file; if no → serves original

Pros: No JavaScript needed, fastest method
Cons: ONLY works on Apache. Does nothing on Nginx.
```

**Method 3: Nginx rewrite rules (manual — Nginx only)**

If using Nginx, add this to your server block:

```nginx
# Serve WebP images when browser supports it AND .webp file exists
# Add INSIDE your server {} block in Nginx config

location ~* \.(jpe?g|png)$ {
    # Check if .webp version exists (created by EWWW)
    set $webp "";
    if ($http_accept ~* "image/webp") {
        set $webp ".webp";
    }

    # Try: 1) WebP version  2) Original file  3) 404
    try_files $uri$webp $uri =404;

    # Cache headers (match step-04a-browser-caching-nginx.md)
    expires 365d;
    add_header Cache-Control "public, immutable";
    add_header Vary "Accept";  # CRITICAL: tells CDN to cache both versions
}
```

> **⚠️ The `Vary: Accept` header is critical.** Without it, CDN caches only ONE version (WebP or JPEG) and serves it to ALL browsers. With `Vary: Accept`, CDN caches both versions and serves the right one based on the browser's Accept header.

---

### 3.8 — Bulk Optimize Existing Images

After installing EWWW, optimize all existing images in the media library:

```bash
# Option A: Via WP Admin UI
# Media → Bulk Optimize → Start Optimizing
# Shows progress bar and savings per image

# Option B: Via WP-CLI (better for large libraries)
# Optimize all images — runs in background, no timeout risk
wp ewwwio optimize media --noprompt

# Check how many images need optimization
wp ewwwio count
```

**Via Admin (recommended for first time — visual feedback):**

1. Go to **Media → Bulk Optimize**
2. Click **"Scan for unoptimized images"**
3. Review the count (e.g., "4,832 images to optimize")
4. Click **"Start Optimizing"**
5. Let it run — shows savings per image (e.g., "hero-banner.jpg: 2.4MB → 1.8MB, saved 25%")

> **⚠️ For large libraries (10,000+ images):**
> - Bulk optimization uses server CPU for every image
> - Run during **low-traffic hours** (late night / early morning)
> - If the process times out, just restart — EWWW tracks which images are already optimized and skips them
> - Consider running the WP-CLI command via `screen` or `tmux` so it survives SSH disconnection:
>   ```bash
>   screen -S ewww
>   wp ewwwio optimize media --noprompt
>   # Press Ctrl+A then D to detach
>   # Reconnect later: screen -r ewww
>   ```

---

### 3.9 — Verify EWWW Is Working

```bash
# Upload a test image and check optimization
wp media import /tmp/test-photo-3000px.jpg --title="CDN Test Image"

# Check the uploaded image was resized to 1920px
wp eval '
$id = get_posts(["post_type" => "attachment", "posts_per_page" => 1, "orderby" => "date", "order" => "DESC"])[0]->ID;
$meta = wp_get_attachment_metadata($id);
echo "Width: " . $meta["width"] . "px\n";
echo "Height: " . $meta["height"] . "px\n";
echo "Sizes generated: " . implode(", ", array_keys($meta["sizes"])) . "\n";
'
# Expected: Width: 1920px (resized from 3000px)
# Expected: Sizes: thumbnail, medium, large, hero, card

# Check WebP file was generated
ls -la wp-content/uploads/$(date +%Y/%m)/test-photo-3000px*
# Should see: test-photo-3000px-1920x1280.jpg
#             test-photo-3000px-1920x1280.jpg.webp  ← WebP version
```

---

## 🔧 Part 3: Alternative — Custom Code (No Plugin)

> **When to use this instead of EWWW:**
> - You want zero plugin dependencies
> - Your hosting restricts plugins but allows PHP extensions
> - You need custom compression logic (e.g., different quality per image type)
>
> **When NOT to use this:**
> - You want a UI for bulk optimization (use EWWW)
> - You're a junior developer maintaining this alone (plugins are easier to manage)

```php
/**
 * Custom image optimization on upload — no plugin needed
 * Requires: PHP Imagick extension (check with: php -m | grep imagick)
 * Add to functions.php or a custom plugin
 */
add_filter('wp_handle_upload', function($upload) {
    $file = $upload['file'];
    $type = $upload['type'];

    // Only process images
    if (strpos($type, 'image/') !== 0) return $upload;
    if (!extension_loaded('imagick')) return $upload;

    try {
        $image = new Imagick($file);

        // ── Lossless compression ──
        if ($type === 'image/jpeg') {
            $image->setImageCompression(Imagick::COMPRESSION_JPEG);
            $image->setImageCompressionQuality(82);
            $image->stripImage(); // Remove EXIF (saves 20-50KB)
        } elseif ($type === 'image/png') {
            $image->setOption('png:compression-level', '9');
        }
        $image->writeImage($file);

        // ── Generate WebP version alongside original ──
        $webp_path = preg_replace('/\.(jpe?g|png)$/i', '.webp', $file);
        $image->setImageFormat('webp');
        $image->setImageCompressionQuality(80);
        $image->writeImage($webp_path);

        $image->destroy();
    } catch (Exception $e) {
        error_log('Image optimization failed: ' . $e->getMessage());
        // Don't break the upload — return original unoptimized
    }

    return $upload;
}, 10, 1);
```

> **⚠️ Limitations of custom code approach:**
> - No bulk optimization for existing images (need to write a separate script)
> - No admin UI showing savings
> - No undo/restore to original
> - WebP serving still requires Nginx/Apache config (section 3.7 Method 3)
> - You maintain the code — plugin updates handle edge cases for you

---

## ⚠️ Multisite Considerations

> - EWWW Image Optimizer supports **network activation** — all sub-sites share the same settings
> - Bulk optimization runs per-site: go to each sub-site's Media → Bulk Optimize
> - For WP-CLI bulk optimization across all sites:
>   ```bash
>   wp site list --field=url | while read SITE_URL; do
>       echo "Optimizing: $SITE_URL"
>       wp ewwwio optimize media --noprompt --url="$SITE_URL"
>   done
>   ```
> - WebP files follow the same `/uploads/sites/N/` structure — CDN Pull Zone handles this automatically

---

## 📊 Expected Results

| Metric | Before Optimization | After Optimization |
|---|---|---|
| Average JPEG size | 1.5-3MB | 300-600KB (lossless) |
| Average PNG size | 500KB-2MB | 200-800KB (lossless) |
| WebP vs JPEG | — | 25-35% smaller |
| Page weight (image-heavy page) | 8-15MB | 2-4MB |
| PageSpeed "Properly size images" | ❌ Warning | ✅ Passed |
| PageSpeed "Serve images in next-gen formats" | ❌ Warning | ✅ Passed (WebP) |
| PageSpeed "Defer offscreen images" | ❌ Warning | ✅ Passed (lazy loading) |

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | `big_image_size_threshold` set to 1920 | Upload 3000px image → stored dimensions ≤ 1920px |
| 2 | Custom image sizes registered | `wp eval 'print_r(wp_get_registered_image_subsizes());'` |
| 3 | Unnecessary sizes removed | `medium_large` not in the list above |
| 4 | EWWW installed and active | `wp plugin status ewww-image-optimizer` shows "Active" |
| 5 | Server tools available | `which jpegoptim optipng cwebp` → all return paths |
| 6 | EWWW set to Pixel Perfect (lossless) | Settings → EWWW → Optimize tab |
| 7 | New uploads compressed automatically | Upload test image → Media Library shows "Optimized" |
| 8 | WebP generated for new uploads | `.webp` file exists next to uploaded image |
| 9 | WebP served to Chrome | DevTools → Network → filter images → Content-Type: image/webp |
| 10 | JPEG/PNG fallback for Safari | `curl -I` without WebP Accept header → Content-Type: image/jpeg |
| 11 | `loading="lazy"` on images | View page source → `<img>` tags have `loading="lazy"` |
| 12 | First image NOT lazy loaded | First `<img>` on page has `loading="eager"` or no attribute (LCP) |
| 13 | `srcset` present on images | View page source → `<img>` tags have `srcset="..."` |
| 14 | Existing images bulk optimized | Media → Bulk Optimize shows "All images optimized" |
| 15 | PageSpeed "Properly size images" passed | Run PageSpeed Insights → no image warnings |
| 16 | Vary: Accept header on images | `curl -sI image.jpg \| grep -i vary` → includes "Accept" |

---

> ⬅️ [Previous: Step 2 — CDN Provider Setup](step-02-cdn-provider-setup.md) &nbsp;|&nbsp; ➡️ [Next: Step 4 — WordPress CDN Plugin](step-04-wordpress-cdn-plugin.md)

