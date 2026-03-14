# Task 2 — Step 6: Write & Test Media Migration Script

> **Estimated Duration:** 2 days
> **Goal:** Make the CDN aware of all existing media files and update the WordPress database so all asset URLs point to the CDN domain — tested on staging first.

---

## 📋 Project Requirement (Section 3.3)

> **3.3. Migration of existing images**
> - Create a script to migrate existing media files to CDN
> - Ensure redirection of old URLs to CDN addresses
> - Preserve the structure of the WordPress media library

### What This Requirement Actually Means

The word "migrate" is misleading — it does **NOT** necessarily mean "upload files to CDN storage." There are two approaches, and the correct one depends on your CDN type:

| Requirement Line | What It Means (Pull Zone) | What It Would Mean (Push Zone) |
|---|---|---|
| "Migrate existing media files to CDN" | **Warm the CDN cache** — send HTTP requests through CDN so it fetches and caches every file from your server | Actually upload/copy all files to CDN storage |
| "Ensure redirection of old URLs to CDN addresses" | **Rewrite URLs in the database** (`SQL REPLACE`) — NOT HTTP 301 redirects | Same — rewrite URLs in the database |
| "Preserve the structure of the WordPress media library" | Keep `/wp-content/uploads/YYYY/MM/file.jpg` path structure — CDN mirrors it automatically | Replicate the same directory structure on CDN storage |

---

## 🔑 Key Decision: Pull Zone + Cache Warming vs Push Zone Upload

### How a Pull Zone CDN Actually Works

```
Your WordPress Server                           CDN (BunnyCDN)
┌────────────────────┐                    ┌──────────────────────┐
│  /wp-content/      │                    │                      │
│    uploads/        │   CDN fetches      │   Cached copies      │
│      2024/         │ ◄──── on demand ── │   of your files      │
│        01/         │    (automatic)     │   (114 global PoPs)  │
│          photo.jpg │                    │                      │
│          logo.png  │                    │                      │
└────────────────────┘                    └──────────────────────┘
       ▲                                          ▲
       │                                          │
  Files STAY here                          Browser requests go here
  (single source                           (cdn.yourdomain.com)
   of truth)
```

**You don't upload anything.** The CDN pulls files from your server the first time they're requested. After that, the CDN serves cached copies from the nearest PoP (Point of Presence).

### Pull Zone + Cache Warming (✅ Our Approach)

**What it does:**
1. Files stay on your WordPress server (the "origin")
2. A script sends HTTP requests to `cdn.yourdomain.com/every-image.jpg`
3. CDN fetches each file from your origin and caches it
4. Database URLs are rewritten: `yourdomain.com` → `cdn.yourdomain.com`
5. WordPress CDN plugin rewrites any remaining URLs in HTML output on-the-fly

**Why this is the right choice:**

| Advantage | Why It Matters |
|---|---|
| **No data loss risk** | Files never leave your server — CDN is just a cache |
| **Easy rollback** | Revert the SQL REPLACE → back to original URLs instantly |
| **No sync needed** | New uploads automatically available (CDN pulls on first request) |
| **No storage cost** | Pull Zone is traffic-based, not storage-based |
| **Media Library intact** | WordPress admin still shows all files normally |
| **Simple implementation** | 1 SQL script + 1 cache warming script = done |

### Push Zone / Storage Upload (❌ Not Recommended Unless Needed)

**What it does:**
1. Actually uploads all files to CDN storage (like S3 or BunnyCDN Storage)
2. Files exist in two places (or you delete originals to save disk space)
3. Needs ongoing sync for new uploads (plugin or cron job)

**When you WOULD use Push Zone:**

| Scenario | Why Push |
|---|---|
| Server disk space is critically low | Offload files to free up space |
| Serving very large files (video, 100MB+ archives) | Don't want origin bandwidth hit |
| Compliance requires files NOT on your server | Data residency / sovereignty requirements |
| Using WP Offload Media Pro | Plugin handles push + URL rewriting automatically |

> ⚠️ **For this project, we use Pull Zone + Cache Warming.** If you need Push Zone, use the WP Offload Media Pro plugin (covered briefly in section 6.2) instead of the manual script.

---

## 🎯 Objectives (Pull Zone Approach)
- **Warm CDN cache** — script sends requests through CDN for all existing media files
- **Rewrite database URLs** — `SQL REPLACE` to update `wp_posts`, `wp_postmeta`, and `wp_options`
- **Preserve directory structure** — CDN mirrors `/wp-content/uploads/YYYY/MM/` paths automatically
- Handle errors gracefully with logging
- **DO NOT use 301 redirects** — causes infinite loops with Pull Zone CDN (explained in §6.5)
- **Run on staging first — never run directly on production without staging test**

---

## ⚠️ Multisite Considerations

> If running WordPress Multisite, the migration is more complex because:
> - **Subdomain Multisite** (`site1.yourdomain.com`): Each sub-site has a different origin URL. You need to run the migration script per sub-site, or adjust the script to replace ALL sub-site domains.
> - **Subdirectory Multisite** (`yourdomain.com/site1/`): All sub-sites share the same uploads path pattern but in separate `/sites/N/` directories: `/wp-content/uploads/sites/2/2026/03/image.jpg`.
> - The CDN URL rewriting must respect these paths — verify your CDN plugin handles Multisite before proceeding.

```bash
# Check if Multisite — if not, skip this section
wp eval 'echo is_multisite() ? "Multisite: YES" : "Multisite: NO";'

# For Multisite — list all sites and their upload URLs
wp site list --fields=blog_id,url --format=table
```

For **subdomain Multisite**, run the migration per sub-site:
```bash
# Loop through all sites
wp site list --field=url | while read SITE_URL; do
    echo "Migrating: $SITE_URL"
    wp eval-file cdn-migrate.php --url="$SITE_URL"
done
```

---

## 🔧 Detailed Steps

### 6.1 — Backup the Database First

```bash
# ALWAYS back up before running any migration
wp db export backup-pre-cdn-migration-$(date +%Y%m%d-%H%M).sql

# Verify the backup
ls -lh backup-pre-cdn-migration-*.sql
```

---

### 6.2 — WP Offload Media (Push Zone Only — Skip This)

> **⚠️ This section is for Push Zone CDN only. Since we are using Pull Zone, SKIP to §6.3.**
>
> WP Offload Media physically uploads files to cloud storage (S3, Google Cloud, BunnyCDN Storage).
> We don't need this — our Pull Zone CDN fetches files from our origin server automatically.
> This section is kept here only for reference if you ever switch to Push Zone.

```bash
# DO NOT run this for Pull Zone setup
# wp offload-media upload-all --delete-local=false
```

---

### 6.3 — Cache Warming Script (Pre-Load CDN)

> **What is cache warming?** When you first set up a Pull Zone CDN, the CDN has ZERO files cached. The first visitor to request each image will experience a slow load (CDN fetches from origin → caches → serves). Cache warming sends requests for ALL your images through the CDN ahead of time, so every file is already cached before real users arrive.

#### Why Cache Warming Matters

```
WITHOUT cache warming:                    WITH cache warming:
─────────────────────                    ────────────────────
User requests image                      You run warming script BEFORE launch
  → CDN: "Cache MISS"                     → CDN fetches all 5,000 images
  → CDN fetches from origin               → All cached across 114 PoPs
  → 800ms response (slow!)
                                         User requests image
Next user requests same image              → CDN: "Cache HIT"
  → CDN: "Cache HIT"                      → 40ms response (fast!)
  → 40ms response (fast)
```

#### Step A — Generate List of All Media URLs (Including All Sizes)

> **⚠️ Important:** The `guid` field in `wp_posts` only stores the URL of the original uploaded image. But WordPress generates multiple sizes (thumbnail, medium, large, etc.) — we need to warm ALL of them, not just the originals.

```bash
# ── Method 1: Generate all image size URLs from the database ──
# This creates a PHP script that outputs every image URL including all sizes

cat > /tmp/generate-urls.php << 'SCRIPT'
<?php
/**
 * Generate a list of ALL image URLs (originals + all resized versions)
 * Run with: wp eval-file /tmp/generate-urls.php > /tmp/all-media-urls.txt
 */
global $wpdb;

$attachments = $wpdb->get_results(
    "SELECT ID FROM {$wpdb->posts} 
     WHERE post_type = 'attachment' 
     AND post_mime_type LIKE 'image/%'
     ORDER BY ID ASC"
);

$count = 0;
foreach ($attachments as $att) {
    // Get the original/full URL
    $url = wp_get_attachment_url($att->ID);
    if ($url) {
        echo $url . "\n";
        $count++;
    }
    
    // Get all registered size URLs (thumbnail, medium, large, custom sizes)
    $metadata = wp_get_attachment_metadata($att->ID);
    if (!empty($metadata['sizes'])) {
        $upload_dir = wp_get_upload_dir();
        $base_url = trailingslashit($upload_dir['baseurl']);
        $file_dir = trailingslashit(dirname($metadata['file']));
        
        foreach ($metadata['sizes'] as $size_name => $size_data) {
            echo $base_url . $file_dir . $size_data['file'] . "\n";
            $count++;
        }
    }
}

fwrite(STDERR, "Total URLs generated: {$count}\n");
SCRIPT

# Run the script
wp eval-file /tmp/generate-urls.php > /tmp/all-media-urls.txt 2>/tmp/url-gen-log.txt

# Check results
cat /tmp/url-gen-log.txt
# Example: Total URLs generated: 24,160
#          (5,000 images × ~5 sizes each = ~25,000 URLs)

wc -l /tmp/all-media-urls.txt
# 24160 /tmp/all-media-urls.txt

# Preview
head -10 /tmp/all-media-urls.txt
# https://yourdomain.com/wp-content/uploads/2024/01/hero-banner.jpg          (original)
# https://yourdomain.com/wp-content/uploads/2024/01/hero-banner-300x200.jpg  (thumbnail)
# https://yourdomain.com/wp-content/uploads/2024/01/hero-banner-600x400.jpg  (medium)
# https://yourdomain.com/wp-content/uploads/2024/01/hero-banner-1200x800.jpg (large)
# ...
```

> **Why not just use `guid`?** The `guid` field only stores the original URL (e.g., `hero-banner.jpg`). But on the actual site, most `<img>` tags use resized versions (e.g., `hero-banner-600x400.jpg`). If you only warm the originals, visitors will still get cache MISSes on the sizes they actually see.

#### Step B — Send Requests Through CDN to Warm Cache

```bash
#!/bin/bash
# cdn-cache-warm.sh — Pre-load all media files into CDN cache
#
# Usage: bash cdn-cache-warm.sh
# Run AFTER CDN Pull Zone is set up and DNS is working

ORIGIN="https://yourdomain.com"
CDN="https://cdn.yourdomain.com"
URL_FILE="/tmp/all-media-urls.txt"
LOG_FILE="/tmp/cdn-warm-$(date +%Y%m%d-%H%M).log"
PARALLEL=10          # Number of concurrent requests (don't overload origin)
TOTAL=$(wc -l < "$URL_FILE")
COUNT=0
FAIL=0

echo "Cache warming started: $(date)" | tee "$LOG_FILE"
echo "Total files to warm: $TOTAL" | tee -a "$LOG_FILE"
echo "Concurrent requests: $PARALLEL" | tee -a "$LOG_FILE"
echo "---" | tee -a "$LOG_FILE"

# Replace origin domain with CDN domain and send HEAD requests
cat "$URL_FILE" | sed "s|${ORIGIN}|${CDN}|g" | xargs -P "$PARALLEL" -I {} sh -c '
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" -H "Accept: image/webp,image/*" --max-time 30 "{}")
    if [ "$STATUS" = "200" ]; then
        echo "OK  $STATUS {}"
    else
        echo "FAIL $STATUS {}"
    fi
' 2>&1 | tee -a "$LOG_FILE" | while read line; do
    COUNT=$((COUNT + 1))
    if echo "$line" | grep -q "^FAIL"; then
        FAIL=$((FAIL + 1))
    fi
    # Progress every 100 files
    if [ $((COUNT % 100)) -eq 0 ]; then
        echo "Progress: $COUNT / $TOTAL"
    fi
done

echo "---" | tee -a "$LOG_FILE"
echo "Cache warming completed: $(date)" | tee -a "$LOG_FILE"
echo "Check log: $LOG_FILE"

# Count failures
FAIL_COUNT=$(grep -c "^FAIL" "$LOG_FILE")
echo "Failed requests: $FAIL_COUNT"
if [ "$FAIL_COUNT" -gt 0 ]; then
    echo "⚠️  Review failed URLs:"
    grep "^FAIL" "$LOG_FILE" | head -20
fi
```

> **⚠️ Important Notes:**
> - Set `PARALLEL=10` or lower — too many concurrent requests can overload your origin server
> - Run during **low-traffic hours** (CDN fetch = extra load on origin)
> - The `-H "Accept: image/webp"` header tells BunnyCDN to cache WebP variants too
> - Files that return non-200 may have been deleted from WordPress but still in the database — that's normal
> - **Run time estimate:** ~5,000 images at 10 parallel = ~8-10 minutes

#### Step C — Verify CDN Cache Is Warm

```bash
# Pick 5 random images and check CDN headers
shuf -n 5 /tmp/all-media-urls.txt | sed "s|${ORIGIN}|${CDN}|g" | while read url; do
    echo "--- $url ---"
    curl -sI "$url" | grep -iE "^(x-cache|cdn-cache|cf-cache|age:|content-type)"
    echo ""
done

# Expected output for each:
# x-cache: HIT       ← CDN served from cache (good!)
# cdn-cache: HIT     ← BunnyCDN-specific header
# age: 342           ← Seconds since cached (any number > 0 = warm)
```

> If you see `x-cache: MISS` on files you already warmed, the CDN PoP you're hitting may be different from the one that was warmed. This is normal — CDN has 100+ PoPs globally, and each caches independently. The warming script pre-loads the PoP closest to your server; other PoPs will cache on first real user request.

---

### 6.4 — Database URL Rewriting Script

For a **Pull Zone CDN** (BunnyCDN Pull Zone), files don't need to be physically uploaded — the CDN fetches from origin automatically. The key task is **URL replacement in the database**.

#### Create a WP-CLI Command Script

Save as `cdn-migrate.php` and run with `wp eval-file`:

> ⚠️ **RISK (rewritten from original):** The original script used `get_posts(['posts_per_page' => -1])` which loads ALL posts into memory at once. On a large site with 50,000+ posts, this will **crash PHP with a fatal memory error**, leaving your database half-migrated. The original also used `wp_update_post()` which triggers revision creation, `save_post` hooks, and cache flushes for every single post — making a 30-second migration take hours with unintended side effects.
>
> The corrected script below uses direct `$wpdb` queries for bulk replacement — no memory issues, no hook side effects, completes in seconds.

```php
<?php
/**
 * CDN URL Migration Script (Safe for Large Sites)
 * Run with: wp eval-file cdn-migrate.php
 *
 * What it does:
 * 1. Replaces origin upload URLs with CDN URLs directly in MySQL (no PHP memory issues)
 * 2. Updates wp_posts, wp_postmeta, and wp_options in bulk SQL queries
 * 3. Skips critical options (siteurl, home, active_plugins) to avoid breaking WordPress
 * 4. Logs all changes
 *
 * Why direct SQL instead of wp_update_post():
 * - wp_update_post() creates a revision for EVERY post (doubles your DB rows)
 * - wp_update_post() fires save_post hooks (SEO plugins recalculate, emails trigger, etc.)
 * - wp_update_post() updates post_modified date (messes up "recently modified" queries)
 * - Direct SQL does the replacement instantly with zero side effects
 */

$origin_url = 'https://yourdomain.com/wp-content/uploads';
$cdn_url    = 'https://cdn.yourdomain.com/wp-content/uploads';
$log_file   = WP_CONTENT_DIR . '/cdn-migration-log-' . date('Ymd-Hi') . '.txt';

$log = fopen( $log_file, 'w' );
fwrite( $log, "CDN Migration Started: " . date('Y-m-d H:i:s') . "\n\n" );

global $wpdb;

// ─── 1. Update post content (wp_posts) — single SQL query ─────────────────
echo "Step 1: Updating wp_posts.post_content...\n";

$result1 = $wpdb->query(
    $wpdb->prepare(
        "UPDATE {$wpdb->posts}
         SET post_content = REPLACE(post_content, %s, %s)
         WHERE post_content LIKE %s",
        $origin_url,
        $cdn_url,
        '%' . $wpdb->esc_like( $origin_url ) . '%'
    )
);
echo "  → Updated {$result1} posts\n";
fwrite( $log, "Updated {$result1} posts in wp_posts\n" );

// ─── 2. Update post excerpts (wp_posts.post_excerpt) ──────────────────────
echo "Step 2: Updating wp_posts.post_excerpt...\n";

$result1b = $wpdb->query(
    $wpdb->prepare(
        "UPDATE {$wpdb->posts}
         SET post_excerpt = REPLACE(post_excerpt, %s, %s)
         WHERE post_excerpt LIKE %s",
        $origin_url,
        $cdn_url,
        '%' . $wpdb->esc_like( $origin_url ) . '%'
    )
);
echo "  → Updated {$result1b} post excerpts\n";
fwrite( $log, "Updated {$result1b} post excerpts\n" );

// ─── 3. Update post meta (wp_postmeta) ────────────────────────────────────
echo "Step 3: Updating wp_postmeta...\n";

$result2 = $wpdb->query(
    $wpdb->prepare(
        "UPDATE {$wpdb->postmeta}
         SET meta_value = REPLACE(meta_value, %s, %s)
         WHERE meta_value LIKE %s",
        $origin_url,
        $cdn_url,
        '%' . $wpdb->esc_like( $origin_url ) . '%'
    )
);
echo "  → Updated {$result2} postmeta rows\n";
fwrite( $log, "Updated {$result2} postmeta rows\n" );

// ─── 4. Update wp_options (theme mods, widgets, customizer, etc.) ─────────
echo "Step 4: Updating wp_options...\n";

$result3 = $wpdb->query(
    $wpdb->prepare(
        "UPDATE {$wpdb->options}
         SET option_value = REPLACE(option_value, %s, %s)
         WHERE option_value LIKE %s
           AND option_name NOT IN ('siteurl', 'home', 'active_plugins', 'template', 'stylesheet')",
        $origin_url,
        $cdn_url,
        '%' . $wpdb->esc_like( $origin_url ) . '%'
    )
);
echo "  → Updated {$result3} option rows\n";
fwrite( $log, "Updated {$result3} option rows\n" );

// ─── 5. Flush caches ──────────────────────────────────────────────────────
wp_cache_flush();
echo "Step 5: WordPress object cache flushed\n";

fwrite( $log, "\nMigration Completed: " . date('Y-m-d H:i:s') . "\n" );
fclose( $log );

echo "\n✅ Migration complete! Log saved to: {$log_file}\n";
echo "⚠️  Remember to also flush your page cache (WP Rocket / FastCGI) after migration.\n";
```

---

### 6.5 — Run the Script on Staging

```bash
# SSH into staging server
ssh user@staging-server

cd /var/www/staging-html

# Run the migration script (dry run check first)
wp eval 'echo "Origin posts with old URLs: " . $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_content LIKE \"%yourdomain.com/wp-content/uploads%\""); echo "\n";'

# If count looks correct, run the actual migration
wp eval-file cdn-migrate.php

# Verify 0 posts still have old URL
wp eval 'echo $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_content LIKE \"%yourdomain.com/wp-content/uploads%\""); echo "\n";'
# Expected output: 0
```

---

### 6.6 — Handle Old URLs (DO NOT Use 301 Redirects with Pull Zone CDN)

> 🔴 **CRITICAL RISK: The origin server must NOT redirect `/wp-content/uploads/` to the CDN when using a Pull Zone.**
>
> **Junior explanation:** A Pull Zone CDN works by fetching files from your origin server on demand. Here's the flow:
> 1. Browser asks CDN: "Give me `cdn.yourdomain.com/uploads/image.jpg`"
> 2. CDN doesn't have it → CDN asks your origin: "Give me `yourdomain.com/wp-content/uploads/image.jpg`"
> 3. If your origin 301-redirects back to the CDN → CDN follows the redirect → asks origin again → **INFINITE LOOP** → **502 Bad Gateway** → **ALL images broken site-wide**
>
> **Rule:** Your origin MUST serve the actual files when the CDN requests them. 301 redirects are ONLY safe with a **Push Zone** (where files are physically stored on CDN storage and no longer on your server).

For a Pull Zone CDN, old URLs are handled automatically:
- The CDN plugin (CDN Enabler / WP Rocket) rewrites URLs in the HTML output
- Any external links using old URLs will still work because the files remain on the origin server
- Over time, search engines will discover the CDN URLs from your updated HTML

If you absolutely need to redirect **browser requests only** (not CDN requests), you can use a conditional redirect that checks the User-Agent, but this is complex and generally unnecessary:
```nginx
# ONLY if you specifically need to redirect browser requests (not recommended for Pull Zone)
# This checks that the request is NOT coming from the CDN itself
# location ~* ^/wp-content/uploads/(.*)$ {
#     if ($http_user_agent !~ "BunnyCDN") {
#         return 301 https://cdn.yourdomain.com/wp-content/uploads/$1;
#     }
# }
```

---

### 6.7 — Verify Migration Results on Staging

```bash
# Check no old URLs remain in post content
wp eval 'global $wpdb; echo $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_content LIKE \"%yourdomain.com/wp-content/uploads%\""); echo " posts still have old URLs\n";'

# Check no old URLs remain in postmeta
wp eval 'global $wpdb; echo $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->postmeta} WHERE meta_value LIKE \"%yourdomain.com/wp-content/uploads%\""); echo " postmeta rows still have old URLs\n";'

# Browse staging site and spot-check 10 posts with images
# All image src should point to cdn.yourdomain.com
```

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Database backed up before migration | Backup SQL file exists with today's timestamp |
| 2 | Cache warming script completed | Log file shows total files processed, failures < 1% |
| 3 | CDN returns `x-cache: HIT` for warmed files | `curl -sI` on 5 random CDN URLs shows HIT |
| 4 | URL rewriting script runs without PHP errors | No error output in terminal |
| 5 | 0 posts still contain old origin URLs | `wp eval` query returns 0 |
| 6 | 0 postmeta rows contain old origin URLs | `wp eval` query returns 0 |
| 7 | Migration log file created | `/wp-content/cdn-migration-log-*.txt` exists |
| 8 | Images load on staging after migration | Browse 10+ pages — all images visible |
| 9 | Old URLs still work on origin | `curl -I https://staging.yourdomain.com/wp-content/uploads/sample.jpg` → `200 OK` (file still served from origin — required for Pull Zone CDN) |
| 10 | WordPress Media Library shows CDN URLs | Admin → Media → click image → URL shows cdn.yourdomain.com |
| 11 | No 301 redirects on `/wp-content/uploads/` | `curl -I https://yourdomain.com/wp-content/uploads/any-image.jpg` → `200 OK` (NOT 301/302) |

---

## 📝 Summary: The Complete Migration Process

```
Step 1: Back up database (§6.1)
           │
Step 2: Warm CDN cache — all sizes (§6.3)
           │  ← Send HTTP requests so CDN fetches & caches all images
           │
Step 3: Rewrite URLs in database (§6.4)
           │  ← SQL REPLACE: yourdomain.com → cdn.yourdomain.com
           │
Step 4: Test on staging (§6.5)
           │
Step 5: Verify no 301 redirects exist (§6.6)
           │
Step 6: Verify all images load from CDN (§6.7)
           │
       ✅ Ready for production (Step 7)
```

> **Remember:** With Pull Zone CDN, your files NEVER leave your server. The CDN is a smart cache layer. If anything goes wrong, restoring the database backup instantly reverts everything to pre-migration state.

---

> ⬅️ [Previous: Step 5 — CDN Caching & Security](step-05-cdn-caching-security.md) &nbsp;|&nbsp; ➡️ [Next: Step 7 — Migrate Images to CDN](step-07-migrate-images-to-cdn.md)

