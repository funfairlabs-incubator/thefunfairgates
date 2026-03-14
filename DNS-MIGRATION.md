# FunFairLabs — DNS & Hosting Migration Guide

## Goal

Move `funfairlabs.com` from Squarespace DNS + GAE to **GitHub Pages**, and set up
`classcharts.funfairlabs.com` as a clean subdomain pointing to the existing GAE app.

---

## Current state

| Thing            | Current                          |
|------------------|----------------------------------|
| DNS provider     | Squarespace (Google Domains migrated to Squarespace) |
| Hosting          | Google App Engine (static HTML)  |
| ClassCharts      | `classcharts.appspot.com`        |

---

## Target state

| Domain                          | Hosting              | Repo                          |
|---------------------------------|----------------------|-------------------------------|
| `funfairlabs.com`               | GitHub Pages         | `funfairlabs-incubator/funfairlabs-site` |
| `classcharts.funfairlabs.com`   | GAE (unchanged)      | `funfairlabs-incubator/classcharts-orc`  |
| `vinyl.funfairlabs.com`         | GitHub Pages         | `funfairlabs-incubator/vinyl-vault`      |
| `disc.funfairlabs.com`          | GitHub Pages         | `funfairlabs-incubator/disc-vault`       |
| `expenses.funfairlabs.com`      | GAE                  | TBD                           |
| `life.funfairlabs.com`          | GAE                  | `funfairlabs-incubator/daas` (private)   |
| `leaguepredictions.co.uk`       | Unchanged            | Independent                   |

---

## Step 1 — Set up GitHub Pages on funfairlabs-site

1. In `funfairlabs-incubator/funfairlabs-site` → **Settings → Pages**
2. Source: **GitHub Actions**
3. Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: '.'
      - id: deployment
        uses: actions/deploy-pages@v4
```

4. Add a `CNAME` file in the repo root containing just:
```
funfairlabs.com
```

---

## Step 2 — Update DNS records in Squarespace

Go to **Squarespace Domains → funfairlabs.com → DNS Settings**

### Remove

Remove any existing A records or CNAME for `@` pointing to GAE.

### Add — GitHub Pages A records (apex domain)

| Type | Host | Value              |
|------|------|--------------------|
| A    | @    | 185.199.108.153    |
| A    | @    | 185.199.109.153    |
| A    | @    | 185.199.110.153    |
| A    | @    | 185.199.111.153    |

### Add — www redirect

| Type  | Host | Value                        |
|-------|------|------------------------------|
| CNAME | www  | funfairlabs-incubator.github.io |

### Add — ClassCharts subdomain → GAE

| Type  | Host         | Value               |
|-------|--------------|---------------------|
| CNAME | classcharts  | ghs.googlehosted.com |

### Add — Future GitHub Pages subdomains

| Type  | Host     | Value                               |
|-------|----------|-------------------------------------|
| CNAME | vinyl    | funfairlabs-incubator.github.io     |
| CNAME | disc     | funfairlabs-incubator.github.io     |

> **Note:** GitHub Pages subdomains need their own repos with a `CNAME` file containing the subdomain.
> e.g. `vinyl.funfairlabs.com` → `vinyl-vault` repo has `CNAME` file containing `vinyl.funfairlabs.com`

### Add — GAE app subdomains (when ready)

| Type  | Host     | Value               |
|-------|----------|---------------------|
| CNAME | expenses | ghs.googlehosted.com |
| CNAME | life     | ghs.googlehosted.com |

---

## Step 3 — Map custom domain in GAE (ClassCharts)

```bash
gcloud app domain-mappings create classcharts.funfairlabs.com \
  --project=classcharts
```

GAE will verify DNS and provision an SSL cert automatically (takes ~15 min).

Update `classcharts-orc` environment variables:
```
NEXTAUTH_URL=https://classcharts.funfairlabs.com
```

Update **Google OAuth** → Authorised redirect URIs:
- Add: `https://classcharts.funfairlabs.com/api/auth/callback/google`
- Keep: `https://classcharts.appspot.com/api/auth/callback/google` (until decommissioned)

---

## Step 4 — Enable HTTPS on GitHub Pages

In **Settings → Pages** → tick **Enforce HTTPS** (available once DNS propagates, ~10 min).

---

## Step 5 — Verify

```bash
dig funfairlabs.com
# Should return 185.199.x.x

dig classcharts.funfairlabs.com CNAME
# Should return ghs.googlehosted.com

curl -I https://funfairlabs.com
# HTTP/2 200

curl -I https://classcharts.funfairlabs.com
# HTTP/2 200 (after GAE mapping + SSL provisioning)
```

---

## Notes

- **Squarespace DNS** is fine for this — no need to migrate to Cloudflare unless you want the proxy/firewall features. All records above work directly with Squarespace DNS.
- **TTL**: Squarespace typically sets TTL to 3600s (1hr). DNS changes usually propagate within 30 min in practice.
- **Old GAE app** for the static homepage can be deleted once GitHub Pages is confirmed live — saves a small amount of GAE quota.
- **leaguepredictions.co.uk** — untouched, no changes needed.
