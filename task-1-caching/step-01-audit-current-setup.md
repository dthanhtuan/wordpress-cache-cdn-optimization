# Task 1 — Step 1: Audit Current Setup

> **Estimated Duration:** 0.5 day
> **Goal:** Understand the current server environment and baseline performance before making any changes.

---

## 🎯 Objectives
- Document the current WordPress setup
- Measure baseline performance metrics
- Identify risks and plugin/theme conflicts before configuring caching

---

## 🔧 Detailed Steps

### 1.0 — Install WP-CLI
WP-CLI is the command-line interface for WordPress. It is required for all `wp` commands used in this guide.

```bash
# Download WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

# Verify the download is functional
php wp-cli.phar --info

# Make it executable and move to system path
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp

# Confirm installation
wp --info
````

### 1.1 — Review Installed Plugins & Theme
```bash
# Via WP-CLI — list all active plugins
wp plugin list --status=active

# List active theme
wp theme list --status=active
```
- Manually check for plugins that already handle caching (e.g., W3 Total Cache, WP Super Cache, LiteSpeed Cache)
- If any exist → **deactivate and document** before proceeding to avoid conflicts

---

### 1.2 — Verify Minimum Version Requirements

> ⚠️ **These are hard requirements from the technical spec. Do NOT proceed if any check fails.**

```bash
# Check WordPress version (minimum: 5.8+)
wp core version
# If below 5.8 → STOP. Update WordPress first: wp core update

# Check PHP version (minimum: 7.4+, recommended: 8.1+)
php -v
# If below 7.4 → STOP. PHP 7.4 is required for Redis Object Cache, Imagify, and WP Rocket.
# If 7.4 → works, but note: PHP 7.4 reached End-of-Life in Nov 2022 (no security patches).
# If 8.0+ → good. If 8.1+ → ideal.

# Check PHP-FPM is running (needed for Nginx FastCGI caching in Step 6)
systemctl status php*-fpm
# Note the exact version — you'll need the socket path later:
ls /run/php/   # e.g., php8.1-fpm.sock
```

**Record your versions:**

| Component | Required | Your Version | Pass? |
|-----------|----------|-------------|-------|
| WordPress | 5.8+ | ___ | ✅/❌ |
| PHP | 7.4+ (8.1+ recommended) | ___ | ✅/❌ |
| PHP-FPM | Running | ___ | ✅/❌ |

> 💡 **Junior note — Why PHP 7.4 minimum?**
> - Redis Object Cache plugin v2.1+ requires PHP 7.4 for typed properties
> - Imagify and ShortPixel image optimization plugins dropped PHP 7.3 support
> - WP Rocket 3.12+ requires PHP 7.4
> - WordPress itself requires PHP 7.4+ since WP 6.3
> - PHP 8.1 is ~20% faster than 7.4 for WordPress workloads (JIT compiler)

---

### 1.3 — Review Server Configuration
```bash
# Check web server
nginx -v          # for Nginx
apache2 -v        # for Apache

# Check if Redis or Memcached is already installed
redis-cli ping    # should return PONG if Redis is running
php -m | grep memcached

# Check current Nginx config
cat /etc/nginx/nginx.conf
cat /etc/nginx/sites-enabled/yourdomain.conf
```
- Note down: server type (Nginx/Apache), existing cache modules

---

### 1.4 — Check WordPress Version & Multisite Config
```bash
# Check WordPress version
wp core version

# Check if Multisite is enabled
wp eval 'echo is_multisite() ? "Multisite: YES" : "Multisite: NO";'
```

---

### 1.5 — Measure Baseline Performance
Run the following tools **before** any changes and **save the results**:

| Tool | URL | What to Record |
|------|-----|----------------|
| GTmetrix | https://gtmetrix.com | TTFB, Page Load Time, Performance Score |
| WebPageTest | https://www.webpagetest.org | TTFB, Waterfall chart |
| Google PageSpeed | https://pagespeed.web.dev | Mobile + Desktop Score |

> 💡 **Save screenshots** of the baseline results. You'll need them to prove improvement at the end.

---

### 1.6 — Identify Conflict-Prone Areas
Check for the following known conflict areas:
- [ ] WooCommerce (cart/checkout pages must never be cached)
- [ ] Membership plugins (user-specific content)
- [ ] Contact Form 7 / Gravity Forms (nonce expiry issues with aggressive caching)
- [ ] Page builders (Elementor, Divi) — may conflict with JS minification

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Baseline TTFB recorded | Screenshot from GTmetrix saved |
| 2 | Baseline PageSpeed score recorded | Screenshot from PageSpeed Insights saved |
| 3 | Active plugins documented | `wp plugin list --status=active` output saved |
| 4 | Server type & PHP version confirmed | `nginx -v && php -v` output saved |
| 5 | No conflicting caching plugins active | Plugin list shows no other caching plugins active |
| 6 | Multisite status confirmed | WP-CLI output documents YES or NO |

---

## 📝 Output / Deliverable
- `audit-baseline-report.md` — a short document capturing all findings above
- Baseline screenshots saved in `/docs/baseline/`

---

> ⬅️ [Back to README](../README.md) &nbsp;|&nbsp; ➡️ [Next: Step 2 — Page Caching Plugin](step-02-page-caching-plugin.md)

