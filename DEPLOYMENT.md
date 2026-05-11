# IEC Ezan — Deployment Notes

This folder contains the two sites for the **Islamic & Educational Center Ezan of Greater Des Moines**:

- `iecezan.com/` → main site (apex domain)
- `donate.iecezan.com/` → donation page (subdomain)

Both are static HTML, deployed to GitHub Pages, behind Cloudflare DNS.

---

## 1 · Three GitHub repos, one Cloudflare zone

| Repo | Purpose | GitHub Pages domain |
|---|---|---|
| `iecezan-website` (public) | Main site | `iecezan.com` |
| `iecezan-donate` (public) | Donate subdomain | `donate.iecezan.com` |
| (no repo for icezan.com) | Just a redirect via Cloudflare | n/a |

### Create the main-site repo
1. New GitHub repo → `iecezan-website` (public).
2. Upload everything from this `iecezan.com/` folder (including the `CNAME` file and `assets/` folder).
3. Settings → Pages → Source: `main` branch / root.
4. Custom domain: `iecezan.com` (already in the CNAME file — GitHub will detect it).
5. Enforce HTTPS: check once GitHub finishes provisioning the cert (≈ 10 minutes).

### Create the donate-subdomain repo
1. New GitHub repo → `iecezan-donate` (public).
2. Upload everything from `donate.iecezan.com/` (including its `CNAME` file).
3. Settings → Pages → Source: `main` branch / root.
4. Custom domain: `donate.iecezan.com` (already in the CNAME file).
5. Enforce HTTPS: check once provisioning completes.

---

## 2 · Cloudflare DNS — iecezan.com zone

Log in to Cloudflare, select the `iecezan.com` zone, and add these records.

### A) Apex domain → GitHub Pages
Four A records on the apex (`@`):

| Type | Name | Content | Proxy |
|---|---|---|---|
| A | @ | `185.199.108.153` | DNS only (gray cloud) |
| A | @ | `185.199.109.153` | DNS only (gray cloud) |
| A | @ | `185.199.110.153` | DNS only (gray cloud) |
| A | @ | `185.199.111.153` | DNS only (gray cloud) |

**Important:** Set to "DNS only" (gray cloud) at first while GitHub provisions the SSL cert. Once HTTPS is working in GitHub Pages, you can turn the orange cloud on for Cloudflare proxying.

### B) `www` subdomain → apex
| Type | Name | Content | Proxy |
|---|---|---|---|
| CNAME | www | `<your-github-username>.github.io` | DNS only |

Then in the `iecezan-website` repo Settings → Pages, you can choose to redirect www → apex if desired.

### C) Donate subdomain → GitHub Pages
| Type | Name | Content | Proxy |
|---|---|---|---|
| CNAME | donate | `<your-github-username>.github.io` | DNS only (gray cloud at first) |

Same dance — DNS-only until GitHub provisions the cert, then optional orange cloud.

---

## 3 · Cloudflare DNS — icezan.com zone (the redirect domain)

`icezan.com` is the legacy domain — everything should 301 to `iecezan.com`.

The cleanest way is **Cloudflare Bulk Redirects** (free tier supports this).

### Option A · Bulk Redirects (recommended)

1. In the Cloudflare dashboard, go to **Rules → Bulk Redirects**.
2. Create a **redirect list**:
   - Name: `icezan-to-iecezan`
   - Add a rule: Source URL `https://icezan.com/*` → Target URL `https://iecezan.com/$1`
   - Status: 301 (permanent)
   - Include subdomains: yes
   - Preserve query string: yes
3. Create a **redirect rule** that applies the list:
   - Name: `icezan.com → iecezan.com`
   - Expression: match the list above.
   - Deploy.

### Option B · Single Page Rule (simpler but less flexible)

If you don't want bulk redirects:

1. **Rules → Page Rules** → Create page rule
2. URL match: `*icezan.com/*`
3. Action: Forwarding URL → 301 Permanent Redirect → `https://iecezan.com/$2`

You'll also need the `icezan.com` zone to have a proxied DNS record (any A record will do — Cloudflare itself handles the redirect at the edge). Easiest: add an A record `@` pointing to `192.0.2.1` (placeholder), set to **proxied** (orange cloud). The page rule intercepts the request before it hits that placeholder.

---

## 4 · Cloudflare security headers (do this once both sites are live)

In Cloudflare → **Rules → Transform Rules → Modify Response Header**, create one rule per zone:

| Header | Value |
|---|---|
| Strict-Transport-Security | `max-age=31536000; includeSubDomains; preload` |
| X-Content-Type-Options | `nosniff` |
| X-Frame-Options | `SAMEORIGIN` |
| Referrer-Policy | `strict-origin-when-cross-origin` |
| Permissions-Policy | `camera=(), microphone=(), geolocation=()` |

Also in **SSL/TLS**:
- Always Use HTTPS: on
- Min TLS: 1.2
- Automatic HTTPS Rewrites: on

---

## 5 · Already wired (you don't need to do anything for these)

- **Formspree contact form** → form ID `xzdoklwz` already in `iecezan.com/index.html`. Notifications will go to whatever email you set in your Formspree dashboard. (Should be `ICEzanDSM@gmail.com`.)
- **Givebutter campaign** → iframe pointing at `https://givebutter.com/embed/c/build-the-house-of-allah-islamic-education-center-ezan-mj2i1d`. Fallback button also links to the full GiveButter campaign page.
- **Instagram link** → `https://www.instagram.com/iecezan/` wired in the Social section + footer.
- **Facebook page plugin** → `https://www.facebook.com/ICEzanDSM/` embedded via Facebook Page Plugin iframe + linked in footer.

### Optional later additions

#### Google Analytics 4
Add this snippet right before `</head>` in **both** `index.html` files:

```html
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-XXXXXXXXXX');
</script>
```

#### Switch GiveButter iframe → official widget (smoother UX)
Grab your widget ID from the GiveButter dashboard (Settings → Embed) and swap the iframe block in `donate.iecezan.com/index.html` for:

```html
<script src="https://widgets.givebutter.com/latest.umd.cjs?acpt"></script>
<givebutter-widget id="YOUR_WIDGET_ID"></givebutter-widget>
```

#### Donation thermometer (progress bar)
GiveButter has a goal/thermometer feature built-in — set a goal amount in your campaign settings and the embedded iframe will show progress automatically. If you want a standalone thermometer on the page (separate from the form), the simplest options:
1. Use GiveButter's "Goal Bar" embed widget (Settings → Embed → Goal Bar) — same dashboard, different snippet.
2. Or build a custom one: track raised total in a JS variable and render a styled bar above the form. Easiest is to use GiveButter's goal feature inside the campaign itself.

---

## 6 · Quick checklist before going live

- [ ] Both repos created on GitHub and Pages enabled
- [ ] Main site live at `https://iecezan.com` (resolves at root — no `/index.html` needed)
- [ ] Donate page live at `https://donate.iecezan.com` (also resolves at root)
- [ ] `https://icezan.com` 301-redirects to `https://iecezan.com`
- [ ] `https://www.iecezan.com` resolves
- [ ] EN ↔ BS toggle works on every page (and persists across pages via localStorage)
- [ ] Clicking the logo/name at top of donate page returns to main site
- [ ] Vaktija modal loads and shows today's prayer times
- [ ] Maps embed loads on the main site
- [ ] Facebook page plugin loads on the main site
- [ ] Instagram link opens `@iecezan`
- [ ] Formspree test submission lands in `ICEzanDSM@gmail.com`
- [ ] GiveButter iframe loads and accepts a $1 test donation
- [ ] All phone / email / address details look right
- [ ] Tested on a phone with mobile data (not just wifi)
