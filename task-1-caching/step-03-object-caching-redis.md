# Task 1 — Step 3: Configure Object Caching via Redis or Memcached

> **Estimated Duration:** 1.5 days
> **Goal:** Cache repeated database queries and PHP object results in memory to reduce database load and speed up dynamic pages.

---

## 🎯 Objectives
- Install Redis (or Memcached) on the server
- Connect WordPress to the cache backend
- Set TTL and test cache hits/misses

---

## 📌 Redis vs Memcached — Quick Decision

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Persistence (survives restart) | ✅ Yes | ❌ No |
| Data structures | Rich (lists, sets, hashes) | Simple key-value only |
| WordPress plugin support | ✅ Excellent | ✅ Good |
| Multisite support | ✅ Yes | ✅ Yes |
| **Recommendation** | ✅ **Preferred** | Alternative |

> ✅ Use **Redis** unless Memcached is already provisioned on the server.

---

## 🔧 Detailed Steps

### 3.1 — Install Redis on the Server
```bash
# Ubuntu / Debian
sudo apt update
sudo apt install redis-server -y

# Start and enable Redis on boot
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Test Redis is running
redis-cli ping
# Expected output: PONG
```

---

### 3.2 — Verify Redis is Listening
```bash
# Check Redis is listening on default port 6379
ss -tlnp | grep 6379

# Check Redis config file
cat /etc/redis/redis.conf | grep -E "^bind|^port|^maxmemory"
```

Set a memory limit to prevent Redis from consuming all RAM:
```bash
# Edit /etc/redis/redis.conf
sudo nano /etc/redis/redis.conf
```
Add or update:
```conf
maxmemory 256mb
maxmemory-policy allkeys-lru
```
```bash
# Restart Redis to apply changes
sudo systemctl restart redis-server
```

---

### 3.3 — Install PHP Redis Extension
```bash
# Ubuntu
sudo apt install php-redis -y

# Verify extension is loaded
php -m | grep redis
# Expected output: redis
```

---

### 3.4 — Install WordPress Redis Object Cache Plugin
```bash
# Via WP-CLI
wp plugin install redis-cache --activate
```
Or manually: **Dashboard → Plugins → Add New → search "Redis Object Cache" by Till Krüss → Install → Activate**

---

### 3.5 — Configure wp-config.php
```bash
# Edit wp-config.php
nano /var/www/html/wp-config.php
```
Add the following **above** the `/* That's all, stop editing! */` line:
```php
// Redis Object Cache Configuration
define( 'WP_REDIS_HOST', '127.0.0.1' );
define( 'WP_REDIS_PORT', 6379 );
define( 'WP_REDIS_TIMEOUT', 1 );
define( 'WP_REDIS_READ_TIMEOUT', 1 );
define( 'WP_REDIS_DATABASE', 0 );

// Set TTL for cached objects: 2 hours
define( 'WP_REDIS_MAXTTL', 7200 );

// Prefix — important for Multisite to avoid key collisions
define( 'WP_REDIS_PREFIX', 'yoursite_' );
```

---

### 3.6 — Enable the Object Cache Drop-In
```bash
# Via WP-CLI
wp redis enable

# Confirm the drop-in is installed
wp redis status
```
Expected output:
```
Status:    Connected
Host:      127.0.0.1
Port:      6379
Database:  0
Prefix:    yoursite_
```

---

### 3.7 — Configure Redis Graceful Degradation

> ⚠️ **RISK: If Redis crashes without graceful degradation, WordPress will hang** waiting for the Redis connection timeout on every single request, making your entire site unusably slow. Always enable this:

Add to `wp-config.php`:
```php
// If Redis goes down, WordPress continues working (just slower, hitting MySQL directly)
define( 'WP_REDIS_GRACEFUL', true );

// Uncomment during planned Redis maintenance to bypass Redis entirely:
// define( 'WP_REDIS_DISABLED', true );
```

---

### 3.8 — (Alternative) Memcached Setup
If you choose Memcached instead:
```bash
# Install Memcached
sudo apt install memcached php-memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
```

> ⚠️ **RISK (was incorrect in original doc):** The Memcached drop-in expects a **global variable**, NOT a `define()` constant. Using `define()` silently fails — WordPress will appear to work but object caching won't actually connect.

Add to `wp-config.php` (**above** `/* That's all, stop editing! */`):
```php
// Memcached server configuration — MUST be a global variable, not a constant
global $memcached_servers;
$memcached_servers = array(
    'default' => array( '127.0.0.1:11211' ),
);
```
Install **W3 Total Cache** which supports Memcached as an object cache backend.

> ⚠️ **Correction:** WP Fastest Cache does **NOT** support Memcached as an object cache backend — only W3 Total Cache does from the popular plugins.

---

## ✅ Validation Checklist

| # | Check | How to Verify |
|---|-------|---------------|
| 1 | Redis is running | `redis-cli ping` returns `PONG` |
| 2 | PHP Redis extension loaded | `php -m \| grep redis` returns `redis` |
| 3 | WordPress connected to Redis | `wp redis status` shows `Connected` |
| 4 | Object cache drop-in installed | File exists: `/wp-content/object-cache.php` |
| 5 | Cache hits are occurring | `redis-cli monitor` — reload the site and see keys being set/fetched |
| 6 | TTL is correct (2 hours) | `redis-cli TTL <key>` returns a value ≤ 7200 |
| 7 | Memory limit enforced | `redis-cli info memory \| grep maxmemory` |
| 8 | Site works correctly | Browse the site — no errors, pages load normally |
| 9 | Multisite prefix is set | `redis-cli keys "*yoursite_*"` shows site-specific keys |

---

> ⬅️ [Previous: Step 2 — Page Caching Plugin](step-02-page-caching-plugin.md) &nbsp;|&nbsp; ➡️ [Next: Step 4 — Browser Caching](step-04-browser-caching.md)

