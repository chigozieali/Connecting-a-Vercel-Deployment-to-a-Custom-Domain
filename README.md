# Connecting a Vercel Deployment to a Custom Domain

A step-by-step guide to taking a project from a GitHub repo, deploying it on Vercel, and serving it from your own domain with HTTPS, redirects, and sensible configuration.

---

## What You'll Build

```
GitHub repo ──push──▶ Vercel (auto build + deploy) ──▶ yourdomain.com (HTTPS)
                                                  └──▶ www.yourdomain.com (redirects to apex)
```

## Prerequisites

- A GitHub account and a repo containing your project (static site, Next.js, React, etc.)
- A free Vercel account (sign up with GitHub for the easiest integration)
- A registered domain name (from Namecheap, Cloudflare, GoDaddy, Whogohost, Qservers, etc.)
- Access to the domain's DNS settings

---

## Part 1: Deploy the Project to Vercel

### Step 1: Push your project to GitHub
```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

### Step 2: Import the repo into Vercel
1. Log in at vercel.com.
2. Click **Add New → Project**.
3. Choose **Import Git Repository** and authorize Vercel to access GitHub (you can limit it to selected repos).
4. Select your repo and click **Import**.

### Step 3: Configure build settings
Vercel auto-detects most frameworks. Confirm:

| Setting | Typical value |
|---|---|
| Framework Preset | Auto-detected (Next.js, Vite, Other for plain HTML) |
| Root Directory | `./` (or the subfolder if your project lives in one) |
| Build Command | Framework default (blank for plain HTML) |
| Output Directory | Framework default (`dist`, `build`, `public`, etc.) |

Add any **Environment Variables** now (Settings → Environment Variables later also works).

### Step 4: Deploy
Click **Deploy**. When it finishes you get a URL like `your-project.vercel.app`. Open it and confirm the site works before touching the domain.

---

## Part 2: Add Your Custom Domain in Vercel

### Step 5: Open the Domains settings
Project → **Settings → Domains**.

### Step 6: Add the domain
1. Enter your apex domain, e.g. `yourdomain.com`, and click **Add**.
2. Vercel will recommend adding `www.yourdomain.com` too and redirecting one to the other. Accept this (recommended: `www` redirects to apex, or the reverse; pick one canonical version).
3. Vercel shows the DNS records it needs. **Always use the exact values shown in your dashboard**, as Vercel can provide project-specific values.

Commonly the records are:

| Type | Name/Host | Value |
|---|---|---|
| A | `@` | `76.76.21.21` |
| CNAME | `www` | `cname.vercel-dns.com` |

---

## Part 3: Configure DNS

You have two options. Pick **one**.

### Option A: Keep your registrar's DNS (add records)
Best if you already have email (MX records) or other services on the domain.

1. Log in to your registrar/DNS provider.
2. Open the DNS management page for your domain.
3. Remove conflicting records for `@` and `www` (old A records, parking-page CNAMEs, URL-forwarding rules).
4. Add the A record for `@` and the CNAME record for `www` using the values from Vercel.
5. Leave MX, TXT (SPF/DKIM), and other records alone.
6. Save.

**Cloudflare users:** set the records to **DNS only** (grey cloud) at first. Turning on the orange proxy can interfere with Vercel's certificate issuance and verification.

### Option B: Use Vercel's nameservers
Best if you want Vercel to manage all DNS.

1. In Vercel's Domains page, choose the **Nameservers** method.
2. At your registrar, replace the nameservers with:
   - `ns1.vercel-dns.com`
   - `ns2.vercel-dns.com`
3. Recreate any existing records you still need (MX, TXT) in Vercel under **Domains → your domain → DNS Records**, otherwise email and other services will break.

---

## Part 4: Verify and Wait for Propagation

### Step 7: Check status in Vercel
Back in **Settings → Domains**, the domain shows **Invalid Configuration** until DNS propagates, then flips to **Valid Configuration**. This usually takes minutes but can take up to 24-48 hours.

### Step 8: Verify DNS from your machine
```bash
# Linux / macOS
dig yourdomain.com +short
dig www.yourdomain.com +short

# Windows
nslookup yourdomain.com
nslookup www.yourdomain.com
```
You should see Vercel's IP for the apex and a Vercel hostname for `www`.

You can also check globally with a tool like dnschecker.org.

### Step 9: Confirm HTTPS
Vercel issues a free SSL certificate automatically (Let's Encrypt) once DNS is valid. Visit `https://yourdomain.com` and confirm the padlock appears. No manual certificate work is needed, and renewals are automatic.

---

## Part 5: Configuration Best Practices

### Step 10: Set the production branch
**Settings → Git → Production Branch** should be `main`. Pushes to `main` deploy to your domain; pushes to other branches create preview URLs.

### Step 11: Set up redirects (optional)
Domain-level redirects (e.g. www → apex) are configured in the Domains page. For path-level rules, add a `vercel.json` to the repo root:

```json
{
  "redirects": [
    { "source": "/old-page", "destination": "/new-page", "permanent": true }
  ],
  "cleanUrls": true,
  "trailingSlash": false
}
```

### Step 12: Add security headers (optional but recommended)
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" }
      ]
    }
  ]
}
```
Commit and push; Vercel redeploys automatically.

### Step 13: Environment variables per environment
**Settings → Environment Variables**: scope each variable to Production, Preview, and/or Development. Redeploy after changing them, since they apply only to new builds.

### Step 14: Add subdomains (optional)
To serve another project at `app.yourdomain.com`: add the domain in that project's Settings → Domains, then create a CNAME record for `app` pointing to the value Vercel gives you.

---

## Part 6: Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| "Invalid Configuration" persists | Wrong or conflicting DNS records | Re-check values, delete old A/CNAME records for the same host |
| SSL certificate not issued | DNS not propagated, Cloudflare proxy on, or CAA records blocking | Wait, set Cloudflare to DNS only, ensure CAA allows `letsencrypt.org` |
| Domain already in use | Domain attached to another Vercel project or account | Remove it from the other project first, or verify ownership with the TXT record Vercel asks for |
| Site shows old content | Cached deployment or browser cache | Hard refresh, check the latest deployment is Production and "Ready" |
| 404 on the domain but `.vercel.app` works | Domain not assigned to the correct project | Check Settings → Domains on the right project |
| Email stopped working after nameserver change | MX/TXT records not recreated in Vercel DNS | Re-add them under Vercel DNS Records |
| Build fails | Wrong build command, output directory, or missing env vars | Read build logs under Deployments |

---

## Part 7: Final Checklist

- [ ] Project deploys successfully on `*.vercel.app`
- [ ] Domain added in Vercel (apex + `www`)
- [ ] DNS records created (or nameservers changed)
- [ ] Domain status is **Valid Configuration**
- [ ] HTTPS works on both apex and `www`
- [ ] One canonical version redirects the other
- [ ] Production branch set to `main`
- [ ] Environment variables set for Production
- [ ] `vercel.json` committed (redirects/headers if needed)
- [ ] README updated with live URL and setup notes

---

## Suggested Repo Structure for This Project

```
vercel-custom-domain-project/
├── README.md              # This guide + live URL
├── vercel.json            # Redirects, headers, clean URLs
├── index.html             # (or your framework's source)
├── screenshots/
│   ├── 01-import-repo.png
│   ├── 02-add-domain.png
│   ├── 03-dns-records.png
│   └── 04-valid-config.png
└── .gitignore
```

**Tip:** Add screenshots of each stage (with any sensitive values blurred) to make this a strong portfolio piece, and include the live domain link at the top of the README.

---

## Useful Commands (Vercel CLI)

```bash
npm i -g vercel
vercel login
vercel link              # link local folder to the Vercel project
vercel --prod            # manual production deploy
vercel domains ls        # list domains
vercel domains add yourdomain.com
vercel env pull          # download env vars to .env.local
```
