# Task 2 — Step 1: Audit Current Media Library

> **Estimated Duration:** 0.5 day
> **Goal:** Understand the current state of the WordPress media library before setting up CDN. Know exactly what needs to be migrated.

---

## 🎯 Objectives
- Count total media files and total size
- Identify dominant file formats
- Document current image URL structure
- Assess bandwidth currently consumed

---

## 🔧 Detailed Steps

### 1.1 — Count Media Files & Disk Usage

```bash
# Total number of files in uploads directory
find /var/www/html/wp-content/uploads/ -type f | wc -l

# Breakdown by file type
find /var/www/html/wp-content/uploads/ -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Total disk size of uploads
du -sh /var/www/html/wp-content/uploads/

# Size by file type
find /var/www/html/wp-content/uploads/ -name "*.jpg" | xargs du -sh 2>/dev/null | tail -1
find /var/www/html/wp-content/uploads/ -name "*.png" | xargs du -sh 2>/dev/null | tail -1
```

---

### 1.2 — Audit via WP-CLI

```bash
# Count total attachments in WordPress database
wp post list --post_type=attachment --format=count

# List all attachment URLs (first 50)
wp post list --post_type=attachment --fields=ID,post_title,guid --format=table | head -50

# Count attachments by mime type
wp eval '
$types = wp_count_attachments();
foreach ($types as $mime => $count) {
    echo $mime . ": " . $count . "\n";
}
'
```

---

### 1.3 — Document Current URL Structure

Identify the current pattern of media URLs:
```
Current format:  https://yourdomain.com/wp-content/uploads/YYYY/MM/filename.jpg

Example:
  https://yourdomain.com/wp-content/uploads/2024/01/hero-image.jpg
  https://yourdomain.com/wp-content/uploads/2024/01/hero-image-1920x1080.jpg  (thumbnail)
  https://yourdomain.com/wp-content/uploads/2024/01/hero-image-300x169.jpg    (thumbnail)
```

After CDN, the target format will be:
```
  https://cdn.yourdomain.com/wp-content/uploads/2024/01/hero-image.jpg
```

---

### 1.4 — Record the Audit Findings

Fill in this table:

| Metric | Value |
|--------|-------|
| Total media files | ___ |
| Total uploads size | ___ GB |
| JPEG files (count) | ___ |
| PNG files (count) | ___ |
| GIF files (count) | ___ |
| SVG files (count) | ___ |
| Other formats | ___ |
| Oldest upload year | ___ |
| WordPress uploads path | `/wp-content/uploads/` |
| Current media URL base | `https://yourdomain.com/wp-content/uploads/` |
| Target CDN URL base | `https://cdn.yourdomain.com/wp-content/uploads/` |

---

### 1.5 — Identify Large Files That Need Optimization

```bash
# Find images larger than 1MB (candidates for compression)
find /var/www/html/wp-content/uploads/ -type f \( -name "*.jpg" -o -name "*.png" \) -size +1M \
  | xargs ls -lh | sort -k5 -hr | head -20
```

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Total file count recorded | `find ... | wc -l` output saved |
| 2 | Total disk size recorded | `du -sh` output saved |
| 3 | File type breakdown documented | `uniq -c` output saved |
| 4 | Current URL structure documented | Example URLs recorded |
| 5 | Large files identified (>1MB) | List of files saved for Step 4 optimization |
| 6 | WordPress attachment count matches filesystem | `wp post list` count matches `find` count |

---

## 📝 Output / Deliverable
- `media-audit-report.md` — filled-in table above
- List of large files requiring pre-optimization

---

> ⬅️ [Back to README](../README.md) &nbsp;|&nbsp; ➡️ [Next: Step 2 — CDN Provider Setup](step-02-cdn-provider-setup.md)

