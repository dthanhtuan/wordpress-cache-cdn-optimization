# Task 2 — Step 5: Configure CDN Caching & Security Policies

> **Estimated Duration:** 1 day
> **Goal:** Configure how long CDN caches images, enable HTTP/2, enforce HTTPS, and protect images from hotlinking and unauthorized access.

---

## 🎯 Objectives
- Set CDN cache TTL to 1 month for images
- Configure proper `Cache-Control` headers at CDN edge
- Enable HTTP/2
- Implement hotlinking protection
- Configure watermarking (if required)
- Enforce HTTPS-only delivery

---

## 🔧 Detailed Steps

### 5.1 — Set CDN Cache TTL

#### BunnyCDN:
Pull Zone → **Caching** tab:
```
Cache Expiration Time:    30 days (2592000 seconds)
Browser Cache Time:       7 days
Stale While Revalidate:   ✅ Enable
Stale If Error:           ✅ Enable (serve stale if origin is down)
```

To confirm the header is correct:
```bash
curl -I https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg | grep -i cache-control
# Expected: cache-control: public, max-age=2592000
```

#### Origin Server — Ensure WordPress serves correct headers:
Add to Nginx virtual host:
```nginx
location ~* ^/wp-content/uploads/.*\.(jpg|jpeg|png|gif|webp|svg|ico)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
    add_header Vary "Accept";  # important for WebP content negotiation
}
```

---

### 5.2 — Enable HTTP/2 on CDN

#### BunnyCDN:
Pull Zone → **General** tab:
```
✅ Enable HTTP/2  (usually enabled by default)
✅ Enable HTTP/3 / QUIC  (if available)
```

#### Verify:
```bash
curl --http2 -I https://cdn.yourdomain.com/wp-content/uploads/sample.jpg
# First line should show: HTTP/2 200
```

---

### 5.3 — Enforce HTTPS Only

#### BunnyCDN:
Pull Zone → **General** tab:
```
✅ Force SSL  (redirects HTTP → HTTPS automatically)
```

#### Origin server (Nginx):
```nginx
# Redirect all HTTP to HTTPS
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com cdn.yourdomain.com;
    return 301 https://$host$request_uri;
}
```

---

### 5.4 — Configure Hotlinking Protection

Hotlinking = other websites embedding your CDN images in their pages, using your bandwidth.

#### BunnyCDN:
Pull Zone → **Security** tab → **Anti-Hotlinking**:
```
✅ Enable Hotlink Protection
Allowed Referrers:
  yourdomain.com
  www.yourdomain.com
  staging.yourdomain.com
  (leave blank = block all external referrers)
```

#### Verify Hotlinking Protection:
```bash
# Request with a foreign Referer — should be blocked (403)
curl -I -H "Referer: https://evilsite.com/" https://cdn.yourdomain.com/wp-content/uploads/sample.jpg
# Expected: HTTP/2 403

# Request from your own domain — should work
curl -I -H "Referer: https://yourdomain.com/" https://cdn.yourdomain.com/wp-content/uploads/sample.jpg
# Expected: HTTP/2 200
```

---

### 5.5 — Configure Watermarking (If Required by Client)

#### Option A — BunnyCDN Image Optimizer (if available on your plan):
Pull Zone → **Optimizer** tab → **Watermark**:
```
✅ Enable Watermark
Image:     Upload your watermark PNG (transparent background)
Position:  Bottom-right
Opacity:   50%
Offset:    10px
```

#### Option B — WordPress-side watermarking via plugin:
```bash
wp plugin install image-watermark --activate
```
**Settings → Watermark:**
```
✅ Automatic watermarking on upload
Watermark image:   Upload your logo
Position:          Bottom right
Opacity:           60%
Image types:       jpg, jpeg, png
```

---

### 5.6 — Configure Security Headers on CDN

Add these security headers at the CDN edge (BunnyCDN → Custom Headers):
```
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Referrer-Policy: strict-origin-when-cross-origin
```

---

### 5.7 — Protect Private/Draft Post Images (Image Privacy)

> ⚠️ **REQUIREMENT (Section 4.2):** Images attached to private posts, drafts, or membership-gated content must NOT be publicly accessible on the CDN. By default, WordPress stores ALL uploads in `/wp-content/uploads/` with no access control — anyone with the URL can view any image, even if the post is private.

#### Problem (Junior explanation):
When you upload an image to a draft or private post, WordPress saves it to `/wp-content/uploads/2026/03/secret-image.jpg`. Even though the post isn't published, the image URL is publicly accessible. Anyone who guesses or discovers the URL can view it. After CDN migration, the CDN caches it too — now it's even MORE public (cached on edge servers worldwide).

#### Solution A — Block Direct Browsing of Upload Directories (Minimum)

Add to Nginx config inside the `server {}` block:
```nginx
# Prevent directory listing of uploads (should already be off, but make sure)
location ~* ^/wp-content/uploads/$ {
    autoindex off;
    return 403;
}

# Block access to sensitive file types in uploads (PHP files, SQL dumps, etc.)
location ~* ^/wp-content/uploads/.*\.(php|phtml|php3|php4|php5|pl|py|jsp|asp|sh|cgi|sql)$ {
    deny all;
    return 403;
}
```

#### Solution B — Protect Private Uploads with a WordPress Auth Gate (Membership/Private Content)

If your site has membership content or private posts with images that should not be publicly accessible:

1. Move private uploads to a non-web-accessible directory:
```php
// In a custom plugin or functions.php
// Override upload directory for private post types
add_filter( 'upload_dir', function( $dirs ) {
    if ( isset( $_POST['post_id'] ) ) {
        $post = get_post( $_POST['post_id'] );
        if ( $post && $post->post_status === 'private' ) {
            $dirs['path']    = WP_CONTENT_DIR . '/private-uploads/' . $dirs['subdir'];
            $dirs['url']     = home_url( '/private-media/' . $dirs['subdir'] );
            $dirs['basedir'] = WP_CONTENT_DIR . '/private-uploads';
            $dirs['baseurl'] = home_url( '/private-media' );
        }
    }
    return $dirs;
});
```

2. Serve private images through a PHP auth gate:
```php
// Create a file: /var/www/html/private-media.php
<?php
require_once __DIR__ . '/wp-load.php';

// Only logged-in users with appropriate access can view
if ( ! is_user_logged_in() ) {
    status_header( 403 );
    exit( 'Forbidden' );
}

$file = WP_CONTENT_DIR . '/private-uploads/' . ltrim( $_SERVER['REQUEST_URI'], '/private-media/' );
if ( file_exists( $file ) ) {
    $mime = wp_check_filetype( $file )['type'];
    header( 'Content-Type: ' . $mime );
    header( 'Cache-Control: private, no-cache' ); // NEVER cache private images
    readfile( $file );
    exit;
}
status_header( 404 );
exit;
```

3. Add Nginx rewrite rule:
```nginx
# Route private media through the auth gate
location /private-media/ {
    try_files $uri /private-media.php?$args;
}
```

4. **Exclude private media from CDN** (critical!):
```
# In CDN Enabler → Exclusions:
/private-media/

# In WP Rocket → CDN → Exclude files:
/private-media/
```

> 💡 **Junior note:** The CDN plugin (CDN Enabler / WP Rocket) will only rewrite URLs that match its inclusion pattern. By keeping private images in `/private-media/` instead of `/wp-content/uploads/`, they are automatically excluded from CDN URL rewriting.

---

### 5.8 — Protect Original Full-Resolution Images

> ⚠️ **REQUIREMENT (Section 4.2):** Original full-resolution images should be protected from unauthorized scraping. CDN serves optimized/resized versions; originals stay protected on origin.

#### Option A — Serve Only Resized Versions via CDN (Recommended)

Configure CDN Enabler or WP Rocket to only rewrite URLs for WordPress-generated thumbnail sizes, not originals:

```php
// In functions.php — remove the full-size image from srcset so it's never linked
add_filter( 'max_srcset_image_width', function() {
    return 1920; // Never serve images wider than 1920px via srcset
});
```

Then on origin, restrict direct access to original (non-resized) images that are larger than your max:
```nginx
# Optional: Block direct access to very large original uploads from outside referrers
# This only blocks hotlinkers — your own site and CDN can still access them
location ~* ^/wp-content/uploads/.*\.(jpg|jpeg|png)$ {
    valid_referers server_names yourdomain.com cdn.yourdomain.com staging.yourdomain.com;
    if ($invalid_referer) {
        return 403;
    }
}
```

#### Option B — Token-Based URL Signing (BunnyCDN)

BunnyCDN supports **Token Authentication** for URL signing — only requests with a valid signed token can access the file:

1. BunnyCDN → Pull Zone → **Security** → **Token Authentication**:
```
✅ Enable Token Authentication
Token Key:    [auto-generated — save this securely]
```

2. Generate signed URLs in WordPress:
```php
// Add to functions.php or a custom plugin
function sign_cdn_url( $url, $expiry_hours = 24 ) {
    $token_key     = 'YOUR_BUNNYCDN_TOKEN_KEY'; // Store in wp-config.php, not here
    $expires       = time() + ( $expiry_hours * 3600 );
    $path          = parse_url( $url, PHP_URL_PATH );
    $hashable_url  = $token_key . $path . $expires;
    $token         = hash( 'sha256', $hashable_url );
    return $url . '?token=' . $token . '&expires=' . $expires;
}

// Usage in templates:
$image_url = sign_cdn_url( 'https://cdn.yourdomain.com/wp-content/uploads/2026/03/photo.jpg' );
echo '<img src="' . esc_url( $image_url ) . '">';
```

> 💡 **Junior note:** Token signing means even if someone finds your image URL, they can't access it after the token expires. This is the strongest protection available without requiring WordPress login.

---

### 5.9 — Secure Data Transfer Between Origin and CDN

> ⚠️ **REQUIREMENT (Section 4.2):** All communication between origin and CDN must use HTTPS (TLS 1.2+). No unencrypted image transfer at any point.

#### Verify Origin Pull Uses HTTPS

When you set up the Pull Zone origin URL (Step 2), it must be `https://`:
```
Origin URL:  https://yourdomain.com    ← HTTPS, not HTTP
             ❌ http://yourdomain.com   ← NEVER use this
```

#### BunnyCDN:
Pull Zone → **Origin** tab → verify:
```
Origin URL:        https://yourdomain.com
✅ Verify SSL Certificate (enable this — ensures CDN rejects invalid/expired origin SSL)
Force Origin SSL:  ✅ Enable
```

#### Verify in Nginx (origin must accept HTTPS):
```bash
# Confirm origin serves over HTTPS
curl -I https://yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# Must return HTTP/2 200 (not a redirect to HTTP, not a certificate error)

# Verify TLS version (minimum: TLS 1.2)
curl -v --tls-max 1.1 https://yourdomain.com 2>&1 | grep "SSL connection"
# This should FAIL (TLS 1.1 rejected), confirming TLS 1.2+ is enforced
```

#### Enforce TLS 1.2+ on Origin Nginx:
```nginx
# In nginx.conf or virtual host SSL config
ssl_protocols TLSv1.2 TLSv1.3;    # Reject TLS 1.0 and 1.1
ssl_prefer_server_ciphers on;
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
```

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | CDN cache TTL = 1 month | `curl -I https://cdn.yourdomain.com/image.jpg` → `Cache-Control: max-age=2592000` |
| 2 | HTTP/2 enabled | `curl --http2 -I https://cdn.yourdomain.com/image.jpg` → `HTTP/2 200` |
| 3 | HTTPS enforced | `curl -I http://cdn.yourdomain.com/image.jpg` → `301 redirect to https://` |
| 4 | SSL certificate valid | `https://cdn.yourdomain.com` — no browser warnings |
| 5 | Hotlinking blocked (external) | `curl -H "Referer: https://external.com/"` → `403 Forbidden` |
| 6 | Hotlinking allowed (own domain) | `curl -H "Referer: https://yourdomain.com/"` → `200 OK` |
| 7 | Watermark visible (if required) | Upload test image → view on site → watermark displayed |
| 8 | `Vary: Accept` header present | Needed for WebP content negotiation — `curl -I` shows `vary: Accept` |
| 9 | Upload dir browsing blocked | `curl https://yourdomain.com/wp-content/uploads/` → `403 Forbidden` |
| 10 | PHP files blocked in uploads | `curl https://yourdomain.com/wp-content/uploads/evil.php` → `403 Forbidden` |
| 11 | Private images require auth | Visit private image URL logged out → `403 Forbidden` |
| 12 | Origin pull uses HTTPS | BunnyCDN Origin settings show `https://` + "Verify SSL" enabled |
| 13 | TLS 1.2+ enforced on origin | `curl --tls-max 1.1 https://yourdomain.com` → fails |
| 14 | Security headers present | `curl -I` shows `X-Content-Type-Options: nosniff` |

---

> ⬅️ [Previous: Step 4 — WordPress CDN Plugin](step-04-wordpress-cdn-plugin.md) &nbsp;|&nbsp; ➡️ [Next: Step 6 — Media Migration Script](step-06-media-migration-script.md)

