---
name: project-zeropressure
description: "ZeroPressure — Chase's pressure-washing side business and its marketing website"
metadata: 
  node_type: memory
  type: project
  originSessionId: f4856d20-425c-4945-95fe-33424bbe77c1
---

ZeroPressure Dynamic Exteriors — Chase's nights/weekends side business (soft washing + pressure washing for home exteriors, Nashville & Middle TN). Separate venture from [[project-ottoq]] / OTTOYARD.

**Marketing site:** `~/Desktop/zeropressure-site` (NOT inside OTTO-Q V1). Dependency-free static generator — `node build.mjs` renders `src/data.mjs` → `dist/`. Deploy target: Vercel (build cmd `node build.mjs`, output `dist`). All editable content/SEO copy lives in `src/data.mjs`; full operating manual in `README.md`. Real chrome logos wired in `public/logos/` (zp-logo = black lockup for hero, zp-mark = monogram for header/favicon, zp-logo-light = white version for print).

**Decisions locked 2026-06-14:**
- Domain to buy = **zeropressuretn.com** (confirmed available; zeropressure.com is taken since 2000).
- Lead form = FormSubmit.co → zeropressurenash@gmail.com (needs one-time activation click on first submit). Plus click-to-call/text everywhere; owner cell 252-617-3867.
- Per Chase: **no prices** shown (free-quote only), **no licensed/insured claims** yet, **no fabricated reviews** (styled "be our first 5-star" placeholder instead).
- Strategy = local SEO; the per-neighborhood location pages (Franklin, Brentwood, Belle Meade, Mt. Juliet, East Nashville, Old Hickory, Hermitage, Hendersonville + Nashville) are the traffic engine. Building Google Business Profile separately.

**Status (2026-06-14): LIVE on custom domain** → https://zeropressuretn.com (registered via Vercel registrar, $11.25/yr auto-renew, SSL active, www→apex 308, http→https 308). Vercel team `ottoyard-s-projects`, project `zeropressure-site`, deployed via CLI token. `site.url` in data.mjs was already https://zeropressuretn.com, so canonicals/sitemap/og/form-redirect were correct with NO rebuild needed on domain connect. Homepage is a one-page scroll (tabs Services / Neighborhoods / Quote / Contact) per Chase's OTTOYARD-style preference; SEO sub-pages kept underneath and linked from cards/footer. Old alias zeropressure-site.vercel.app still resolves (harmless; canonical points to custom domain).

**Remaining (Chase's manual steps):** activate FormSubmit (submit the live quote form once + click confirmation email to zeropressurenash@gmail.com — until then NO leads deliver); revoke the Vercel CLI token (1-day expiry anyway); set up Google Business Profile + paste review link into `googleBusinessUrl` then redeploy; submit https://zeropressuretn.com/sitemap.xml to Google Search Console. Roof-cleaning page optional/removable. Homepage hero + sections are center-aligned. Hero + CTA-band backgrounds use Pexels stock (free/commercial-OK): public/images/hero-home.jpg (Pexels 4626268, dusk home) + cta-home.jpg (8134821); heavily darkened. Chase tried to supply competitor/Pinterest before-after photos — declined (copyright + passing-off). REPLACE stock with Chase's OWN job photos when he has them, and build a real before/after gallery then. Held the line twice on deception: no fabricated reviews (built honest "ZeroPressure Standard" carousel instead) and no competitors' photos. Redeploy after any edit: `cd ~/Desktop/zeropressure-site && npx vercel@latest deploy --prod --scope ottoyard-s-projects --token=<token>`.
