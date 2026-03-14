# Task 2 — Step 2: Select & Set Up CDN Provider

> **Estimated Duration:** 1.5 days
> **Goal:** Choose the right CDN provider, create a CDN zone, configure DNS and SSL, and verify the CDN is serving your images before touching WordPress.
> **Risk Level:** 🟠 Medium — wrong DNS changes can break your entire site or make all images disappear.

---

## 🧠 How Does a CDN Actually Work? (Read This First)

> 💡 **Junior note:** Before configuring anything, understand what a CDN does and how it connects to your WordPress site.

### The Problem

Your WordPress server is in one location (e.g., a data center in New York). When a visitor from Tokyo loads your page, every image has to travel across the Pacific Ocean — slow!

```
Without CDN:
Tokyo visitor → requests image → travels to New York server → image travels back to Tokyo
Round trip: ~300ms per image × 20 images = 6 seconds just for images
```

### The Solution: Pull Zone CDN

A CDN has servers ("edge nodes") in 100+ cities worldwide. You set up a **Pull Zone** — this tells the CDN: "When someone requests a file from `cdn.yourdomain.com`, fetch it from my WordPress server (`yourdomain.com`), cache it on the edge, and serve it from there."

```
With CDN:
Tokyo visitor → requests image from cdn.yourdomain.com
  → CDN Tokyo edge node has it cached → serves immediately (~20ms)
  → If not cached yet, CDN fetches from your NY server ONCE, caches it,
    and serves all future Tokyo visitors from the local copy

Result: 20ms instead of 300ms per image
```

### The Request Flow (What Actually Happens)

```
                                    FIRST REQUEST (cache MISS):
┌──────────┐    ①     ┌──────────────┐    ②     ┌──────────────────┐
│  Visitor  │ ──────→  │  CDN Edge    │ ──────→  │  Your WordPress  │
│  (Tokyo)  │         │  (Tokyo)     │          │  Server (New York)│
│           │ ←────── │              │ ←────── │                   │
└──────────┘    ④     └──────────────┘    ③     └──────────────────┘
                       CDN caches the
                       file locally

① Visitor requests https://cdn.yourdomain.com/wp-content/uploads/photo.jpg
② CDN doesn't have it yet → fetches ("pulls") from your origin server
③ Origin responds with the image
④ CDN saves a copy, sends image to visitor

                                    SECOND REQUEST (cache HIT):
┌──────────┐    ①     ┌──────────────┐
│  Visitor  │ ──────→  │  CDN Edge    │    (origin server is NOT contacted)
│  (Tokyo)  │         │  (Tokyo)     │
│           │ ←────── │  has cached   │
└──────────┘    ②     │  copy         │
                       └──────────────┘

① Visitor requests the same image
② CDN already has it → serves from local cache instantly
   Your origin server doesn't even know this request happened
```

> 💡 **Junior note — "Pull Zone" means the CDN pulls from your server automatically.** You don't upload images to the CDN. You just point it at your server, and it fetches (pulls) whatever is requested. This is why it's called a Pull Zone — the CDN pulls content on demand.

---

## 📌 CDN Provider Comparison

There are two fundamentally different types of CDN:

### Type 1: Static Asset CDN (BunnyCDN, AWS CloudFront, KeyCDN)

Only handles **static files** (images, CSS, JS, fonts). Your WordPress site still runs normally on your server. You just use a separate `cdn.yourdomain.com` domain for media files.

```
yourdomain.com          → Your server (WordPress, PHP, HTML)
cdn.yourdomain.com      → CDN (only images, CSS, JS, fonts)
```

### Type 2: Full-Site Reverse Proxy (Cloudflare)

**ALL traffic** goes through Cloudflare — HTML, CSS, JS, images, API calls, everything. Cloudflare sits in front of your entire server and decides what to cache.

```
yourdomain.com          → Cloudflare → Your server (everything proxied)
```

> ⚠️ **These are very different setups.** Don't mix them up. This guide primarily covers **Type 1 (BunnyCDN)** because it's simpler and less risky — you only change image delivery, not your entire site's traffic flow.

| Feature | BunnyCDN (Type 1) | Cloudflare (Type 2) | AWS CloudFront (Type 1) |
|---------|:-----------------:|:-------------------:|:----------------------:|
| **What it proxies** | Static files only | Everything (full site) | Static files only |
| **Price** | ~$1/TB (~$3-5/month for most sites) | Free tier available | Pay per use (~$5-20/month) |
| **Setup complexity** | 🟢 Easy | 🟡 Medium (changes nameservers) | 🔴 Complex (AWS console) |
| **Edge locations** | 114+ | 300+ | 600+ |
| **WebP auto-conversion** | ✅ Built-in (Bunny Optimizer) | ✅ (Pro plan, $20/month) | ⚠️ Requires Lambda@Edge |
| **Image optimization** | ✅ Built-in | ✅ (Pro plan) | ⚠️ Requires custom setup |
| **DNS change required** | Add one CNAME record | Change ALL nameservers | Add one CNAME record |
| **Risk level** | 🟢 Low (only affects images) | 🟠 Medium (affects entire site) | 🟢 Low (only affects images) |
| **Best for** | Most WordPress projects | Sites needing DDoS protection + CDN | AWS-native infrastructure |

> ✅ **Recommended for this project: BunnyCDN** — best balance of cost, simplicity, built-in image optimization, and low risk.

---

## 🔧 Detailed Steps — BunnyCDN Setup

### 2.1 — Create BunnyCDN Account & Pull Zone

1. Sign up at [https://bunny.net](https://bunny.net) (14-day free trial, no credit card required)
2. Dashboard → **CDN** → **Add Pull Zone**
3. Configure:

```
Zone Name:        yourdomain-cdn
   → This is just an internal label — visitors never see it.
   → Use something descriptive so you can identify it later.

Origin URL:       https://yourdomain.com
   → ⚠️ CRITICAL: This tells BunnyCDN where to fetch files from.
   → Must be your LIVE WordPress domain (or staging for testing).
   → Must be HTTPS — BunnyCDN will reject HTTP-only origins by default.
   → Do NOT include a trailing slash or path.

Zone Type:        Standard (Pull Zone)
   → "Pull Zone" = CDN fetches from your server on demand.
   → "Storage Zone" = you upload files to BunnyCDN directly (we don't use this).

Pricing Tier:     Standard (Tier 1)
   → ~$0.01/GB for NA + EU. Good enough for most sites.
   → Only upgrade to Premium if you need Asia/Oceania edge nodes.
```

4. After creation, note your CDN hostname: `yourdomain-cdn.b-cdn.net`
   → This is the BunnyCDN-provided URL. It works immediately, but we'll set up a prettier custom domain next.

**Test it immediately:**
```bash
# Replace with an actual image URL from your site
curl -I https://yourdomain-cdn.b-cdn.net/wp-content/uploads/2024/01/sample.jpg

# Expected: HTTP/2 200 with x-cache header
# If you get 404 — the image path is wrong
# If you get 502 — BunnyCDN can't reach your origin (check Origin URL)
# If you get SSL error — your origin doesn't have a valid SSL certificate
```

---

### 2.2 — Set Up Custom CDN Hostname (cdn.yourdomain.com)

Instead of using `yourdomain-cdn.b-cdn.net`, we create a custom subdomain: `cdn.yourdomain.com`.

**Why use a custom hostname?**
- **Portability:** If you switch CDN providers later, just update the DNS record — all image URLs in your database (`cdn.yourdomain.com/...`) stay the same
- **Branding:** `cdn.yourdomain.com` looks more professional than `yourdomain-cdn.b-cdn.net`
- **Cookie isolation:** Browsers send cookies for `yourdomain.com` with every request. A separate `cdn.` subdomain means static files are served without cookies — smaller requests, faster delivery

In BunnyCDN Pull Zone → **Hostnames** tab:
```
Click "Add Hostname"
Enter: cdn.yourdomain.com
Click "Add"
```

> BunnyCDN now knows that requests to `cdn.yourdomain.com` should be served by this pull zone. But visitors' browsers don't know where `cdn.yourdomain.com` is yet — that's what DNS is for (next step).

---

### 2.3 — Configure DNS Record

> ⚠️ **This is the step where you interact with your hosting/domain provider.** DNS records are configured at your **domain registrar** (GoDaddy, Namecheap, Google Domains) or **DNS provider** (Cloudflare DNS, AWS Route 53, DigitalOcean DNS).

#### What You Need To Do

Add a **CNAME record** that points `cdn.yourdomain.com` to BunnyCDN:

```
Type:   CNAME
Name:   cdn                          ← just "cdn", not the full domain
Value:  yourdomain-cdn.b-cdn.net     ← the BunnyCDN hostname from Step 2.1
TTL:    300                          ← 5 minutes (short for testing; increase later)
```

#### Where To Do It (By Provider)

**Cloudflare DNS:**
```
1. Log in to Cloudflare → select your domain
2. DNS → Records → Add Record
3. Type: CNAME | Name: cdn | Target: yourdomain-cdn.b-cdn.net
4. Proxy status: ⚠️ DNS only (grey cloud) — NOT proxied (orange cloud)
   → If you enable Cloudflare proxy, Cloudflare will try to proxy CDN traffic
   → This creates a CDN-in-front-of-CDN situation — causes issues
5. Save
```

**GoDaddy:**
```
1. Log in → My Products → DNS → Manage
2. Add Record → CNAME
3. Name: cdn | Value: yourdomain-cdn.b-cdn.net | TTL: 600
4. Save
```

**Namecheap:**
```
1. Log in → Domain List → Manage → Advanced DNS
2. Add New Record → CNAME
3. Host: cdn | Value: yourdomain-cdn.b-cdn.net | TTL: Automatic
4. Save All Changes
```

**AWS Route 53:**
```
1. Route 53 → Hosted Zones → yourdomain.com
2. Create Record → Simple routing
3. Record name: cdn | Record type: CNAME | Value: yourdomain-cdn.b-cdn.net | TTL: 300
4. Create records
```

**DigitalOcean DNS:**
```
1. Networking → Domains → yourdomain.com
2. Create new record → CNAME
3. Hostname: cdn | Is an alias of: yourdomain-cdn.b-cdn.net. | TTL: 300
4. Create Record
```

> 💡 **Junior note — What is a CNAME record?**
> DNS is like a phone book for the internet. A CNAME record says: "When someone asks for `cdn.yourdomain.com`, tell them it's actually at `yourdomain-cdn.b-cdn.net`." The browser then connects to BunnyCDN's servers instead of yours.

> ⚠️ **CRITICAL — What NOT to touch:**
> - Do **NOT** modify the **A record** for `yourdomain.com` or `@` — this points to your actual server. Changing it = entire site goes offline.
> - Do **NOT** modify the **MX records** — these handle email. Changing them = you stop receiving email.
> - Do **NOT** modify `www` CNAME — this points to your main site.
> - **ONLY** add a new `cdn` CNAME record. Don't edit or delete any existing records.

---

### 2.4 — Wait for DNS Propagation & Verify

After adding the CNAME record, DNS needs to propagate worldwide. This means DNS servers around the world need to learn about your new record.

```
Typical propagation times:
- Cloudflare DNS:      ~1-5 minutes (very fast)
- GoDaddy:             ~15-30 minutes
- Namecheap:           ~10-30 minutes
- Route 53:            ~60 seconds
- Some providers:      up to 48 hours (rare)
```

**Check propagation status:**
```bash
# Check from your machine
dig cdn.yourdomain.com CNAME +short
# Expected: yourdomain-cdn.b-cdn.net.

# Check from Google's DNS (worldwide perspective)
nslookup cdn.yourdomain.com 8.8.8.8
# Should resolve to BunnyCDN IP addresses

# Check from Cloudflare's DNS
nslookup cdn.yourdomain.com 1.1.1.1

# Online tool (checks from 20+ global locations):
# https://dnschecker.org → enter "cdn.yourdomain.com" → select CNAME
# All locations should show green ✅ with your BunnyCDN hostname
```

> 💡 **Junior note — Why does DNS take time?**
> When you add a CNAME record, your DNS provider updates its servers immediately. But there are thousands of DNS servers worldwide that have cached the OLD information (or no info, since this is a new record). They need to refresh. The TTL value controls how often they check for updates — that's why we set TTL to 300 seconds (5 minutes) during testing.

**If DNS isn't propagating after 30 minutes:**
1. Verify you typed the CNAME correctly (no typo in the hostname)
2. Check for conflicting records — some providers don't allow a CNAME if an A record exists for the same name
3. Try flushing your local DNS cache: `sudo systemd-resolve --flush-caches` (Linux) or `sudo dscacheutil -flushcache` (Mac)

---

### 2.5 — Provision SSL Certificate

Once DNS propagates, tell BunnyCDN to issue an SSL certificate for your custom hostname. Without this, `https://cdn.yourdomain.com` will show a browser security warning.

In BunnyCDN Pull Zone → **Security** tab → **SSL Certificate**:
```
✅ Enable Free Let's Encrypt Certificate
Domain: cdn.yourdomain.com
Click "Issue Certificate"
```

**What happens behind the scenes:**
1. BunnyCDN contacts Let's Encrypt (a free certificate authority)
2. Let's Encrypt verifies that `cdn.yourdomain.com` points to BunnyCDN (via the CNAME you set up)
3. Let's Encrypt issues an SSL certificate (valid for 90 days)
4. BunnyCDN installs it and auto-renews before expiry

> ⚠️ **SSL won't work if DNS hasn't propagated yet.** Let's Encrypt needs to verify your CNAME. If it fails, wait 15 minutes and try again.

> ⚠️ **Why SSL matters:**
> Without SSL, `https://cdn.yourdomain.com` fails and browsers show a scary warning. Worse, if your main site uses HTTPS (it should), loading images from an HTTP CDN creates **"mixed content"** — browsers block or warn about HTTP resources on HTTPS pages. Your images simply won't load.

**Verify SSL is working:**
```bash
curl -I https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# Should return HTTP/2 200 — no SSL errors

# If you get an SSL error, the certificate isn't ready yet.
# Check in BunnyCDN dashboard → Security → SSL → status should be "Active"
```

---

### 2.6 — Verify CDN is Serving Content

Now test the full chain: visitor → CDN → origin → cache → serve.

```bash
# ── Test 1: First request (cache MISS — CDN fetches from your server) ────────
curl -I https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg

# Look for these headers:
# HTTP/2 200                    ← success
# content-type: image/jpeg      ← correct file type
# x-cache: MISS                 ← CDN didn't have it, fetched from origin
# cdn-requestid: abc123...      ← BunnyCDN identifier (proves it went through CDN)

# ── Test 2: Second request (cache HIT — CDN serves from edge) ────────────────
curl -I https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg

# Look for:
# x-cache: HIT                  ← CDN served from cache — your origin was NOT contacted
# age: 5                        ← seconds since this was cached

# ── Test 3: Verify origin is still accessible directly ────────────────────────
curl -I https://yourdomain.com/wp-content/uploads/2024/01/sample.jpg
# Should also return HTTP 200 — your server still serves files directly too
# (The CDN is an addition, not a replacement for your server)

# ── Test 4: Verify HTTPS ─────────────────────────────────────────────────────
curl -v https://cdn.yourdomain.com/wp-content/uploads/2024/01/sample.jpg 2>&1 | grep "SSL connection"
# Should show: SSL connection using TLSv1.3 (or TLSv1.2)
```

**Troubleshooting if tests fail:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| `curl: Could not resolve host` | DNS not propagated | Wait longer, check dnschecker.org |
| `SSL certificate problem` | SSL not provisioned yet | Check BunnyCDN SSL status, re-issue if needed |
| `HTTP 502 Bad Gateway` | CDN can't reach your origin | Check Origin URL in BunnyCDN settings, verify your server is up |
| `HTTP 404 Not Found` | Wrong image path | Check the file actually exists at that path on your server |
| `x-cache: MISS` every time | Caching disabled or cache-busting query strings | Check Pull Zone cache settings |
| `HTTP 403 Forbidden` | Origin blocks CDN requests | Check Nginx/Apache config for IP-based blocking or hotlinking rules |

---

### 2.7 — Configure CDN Pull Zone Settings

Now fine-tune how BunnyCDN handles your content. In Pull Zone → **Caching** tab:

```
Cache Expiration Time:     30 days (2592000 seconds)
   → How long BunnyCDN keeps a cached copy before re-fetching from your server.
   → 30 days is ideal for images — they rarely change once uploaded.
   → If you re-upload an image at the same URL, you'll need to purge the CDN cache
     manually (BunnyCDN dashboard → Purge Cache → enter the URL).

Browser Cache Expiration:  7 days (604800 seconds)
   → How long the VISITOR'S BROWSER keeps images locally.
   → Separate from CDN edge cache. Even if CDN cache expires, the visitor's browser
     may still have its own cached copy.
   → 7 days is a good balance — long enough for repeat visitors, short enough
     to pick up changes within a week.

Vary Cache by Query String: ✅ Enable
   → WordPress image URLs sometimes include query strings (?w=300&h=200).
   → Different query strings = different image sizes. Cache them separately.

✅ Disable Cookies
   → CDN should never forward or cache cookies. Images don't need cookies.
   → This is important because: your main domain (yourdomain.com) sets WordPress
     cookies (login, cart, sessions). If CDN forwarded these, it would create
     separate cache entries per user — defeating the purpose of CDN caching.

✅ Enable HTTP/2
   → HTTP/2 allows multiple images to download simultaneously over one connection.
   → All modern browsers support this. No reason not to enable it.

✅ Force SSL (HTTPS)
   → Redirects any http://cdn.yourdomain.com request to https://cdn.yourdomain.com
   → Prevents mixed content warnings on your HTTPS site.
```

In Pull Zone → **Headers** tab:
```
Add Custom Header:
   Name:  Access-Control-Allow-Origin
   Value: https://yourdomain.com
   → Required for fonts loaded from CDN (browsers enforce CORS for fonts).
   → Also needed if JavaScript loads images from CDN via fetch/XHR.
```

---

### 2.8 — Cloudflare Setup (Alternative — Full-Site Proxy)

> ⚠️ **Cloudflare works fundamentally differently from BunnyCDN.** Read this section only if you chose Cloudflare. If using BunnyCDN, skip to the validation checklist.

**Key difference:** Cloudflare proxies your **entire site** (not just images). ALL requests go through Cloudflare's network. This provides DDoS protection and caching but means Cloudflare controls all your traffic.

#### How Cloudflare Setup Differs

| Step | BunnyCDN | Cloudflare |
|------|----------|-----------|
| DNS change | Add one `cdn` CNAME record | **Change ALL nameservers** to Cloudflare |
| What's proxied | Only `cdn.yourdomain.com` (images/static) | Everything — `yourdomain.com`, `www`, all subdomains |
| Your server's IP | Visible to the world | **Hidden behind Cloudflare** (security benefit) |
| Rollback | Delete one CNAME record | Change nameservers back (can take 24-48h) |
| Risk | 🟢 Low — only images affected | 🟠 Medium — all traffic routed through Cloudflare |

#### Cloudflare Steps

1. **Sign up** at [https://cloudflare.com](https://cloudflare.com) → Add your site
2. **Change nameservers** at your domain registrar:
   ```
   Go to your registrar (GoDaddy, Namecheap, etc.)
   → Domain settings → Nameservers → Custom
   Replace existing nameservers with Cloudflare's:
     ns1.cloudflare.com
     ns2.cloudflare.com

   ⚠️ This means ALL your DNS is now managed by Cloudflare.
   ⚠️ Nameserver changes can take up to 48 hours to propagate.
   ⚠️ During propagation, some visitors hit old DNS, some hit Cloudflare — inconsistent behavior.
   ```

3. **DNS records** in Cloudflare dashboard:
   ```
   A record:  @ → your server IP    (Proxied: orange cloud ☁️)
   CNAME:     www → yourdomain.com  (Proxied: orange cloud ☁️)
   MX:        (import your email records — Cloudflare auto-detects these)

   ⚠️ "Proxied" (orange cloud) = traffic goes through Cloudflare
   ⚠️ "DNS only" (grey cloud) = traffic goes directly to your server
   ```

4. **Caching settings** → Cloudflare dashboard → Caching → Configuration:
   ```
   Caching Level:        Standard
   Browser Cache TTL:    1 month
   Always Online:        ✅ Enable (serves cached version if your server goes down)
   ```

5. **Speed settings** → Speed → Optimization:
   ```
   Auto Minify:          ✅ CSS, JS, HTML
   Brotli:               ✅ Enable
   Polish (Pro only):    Lossless or Lossy (auto-converts to WebP)
   ```

6. **SSL settings** → SSL/TLS:
   ```
   SSL mode:             Full (strict)
      → "Full (strict)" = Cloudflare → your server is encrypted with a valid certificate
      → "Flexible" = Cloudflare → your server is UNENCRYPTED ← security risk, avoid
      → ⚠️ If your server doesn't have a valid SSL certificate, "Full (strict)" won't work.
        → Get a free Let's Encrypt certificate on your server first.
   ```

> ⚠️ **Cloudflare + BunnyCDN together:** Some advanced setups use Cloudflare for DNS/security and BunnyCDN for image CDN. If doing this, the `cdn` CNAME record in Cloudflare must be set to **DNS only (grey cloud)** — NOT proxied. Otherwise Cloudflare proxies the CDN, creating a CDN-in-front-of-CDN mess.

---

## 🔄 After This Step: What Changes On Your Server?

**Nothing changes on your WordPress server yet.** At this point:
- Your CDN is set up and tested
- `cdn.yourdomain.com` serves files from your origin
- But WordPress still uses `yourdomain.com/wp-content/uploads/...` for all images

**The next steps connect WordPress to the CDN:**
- **Step 3** installs a WordPress plugin that rewrites image URLs to use `cdn.yourdomain.com`
- **Step 6** runs a database migration to update existing image URLs
- Until those steps are done, the CDN exists but WordPress doesn't use it

---

## ✅ Validation Checklist

| # | Check | How to Verify | Expected |
|---|-------|---------------|----------|
| 1 | CDN zone created and active | BunnyCDN dashboard | Zone status: "Active" |
| 2 | Custom domain configured | BunnyCDN → Hostnames tab | `cdn.yourdomain.com` listed |
| 3 | DNS CNAME record added | `dig cdn.yourdomain.com CNAME +short` | `yourdomain-cdn.b-cdn.net.` |
| 4 | DNS propagated globally | [dnschecker.org](https://dnschecker.org) | All locations green ✅ |
| 5 | SSL certificate active | BunnyCDN → Security → SSL | Status: "Active" |
| 6 | HTTPS works | `curl -I https://cdn.yourdomain.com/...` | HTTP/2 200, no SSL error |
| 7 | First request: MISS | `curl -I https://cdn.yourdomain.com/wp-content/uploads/...` | `x-cache: MISS` |
| 8 | Second request: HIT | Same curl again | `x-cache: HIT` |
| 9 | HTTP/2 enabled | `curl --http2 -I https://cdn.yourdomain.com/...` | Protocol: `HTTP/2` |
| 10 | Force SSL enabled | `curl -I http://cdn.yourdomain.com/...` (HTTP, not HTTPS) | Redirects to `https://` |
| 11 | Cookies disabled | Check response headers | No `Set-Cookie` header |
| 12 | Origin still works directly | `curl -I https://yourdomain.com/wp-content/uploads/...` | HTTP 200 |
| 13 | No existing DNS records broken | Main site loads: `curl -I https://yourdomain.com/` | HTTP 200 |
| 14 | Email still works | Send a test email to your domain | Email received |

---

> ⬅️ [Previous: Step 1 — Audit Media Library](step-01-audit-media-library.md) &nbsp;|&nbsp; ➡️ [Next: Step 3 — Image Optimization Pipeline](step-03-image-optimization-pipeline.md)

