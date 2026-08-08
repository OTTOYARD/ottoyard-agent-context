---
name: reference-ottoyard-brand
description: OTTOYARD brand/design system (from the website brand guide) — apply to OTTO-PULSE + OrchestraAV
metadata: 
  node_type: memory
  type: reference
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

OTTOYARD design system, extracted from `~/Desktop/OTTOYARD-Brand-Guide.html` (1.3MB; tokens below). Use this to reskin both Lovable apps (OTTO-PULSE = ottoyard-field-ops, OrchestraAV = ottoyard-a4359174) for the UI audit ([[feedback-ui-audit-branding]]). Both currently use generic shadcn themes — reskin to these tokens.

**Theme = dark, high-contrast, premium, techy, red-accented.**

Colors (CSS vars from the guide):
- Backgrounds: `--bg-0 #06070A` (near-black base), `--bg-1 #0A0B0E`, `--bg-2 #111317` (cards/panels).
- Ink/text: `--ink #E7EAF0` (near-white), `--ink-dim #8A8F99` (secondary), `--ink-faint #4A4E57` (muted).
- Brand red (primary accent): `--red #C8102E`, `--red-hot #E8293F` (hover/active), `--red-deep #8E0B20` (pressed/deep).
- Lines/borders: `rgba(255,255,255,0.06)` (subtle), `rgba(255,255,255,0.12)` (strong).
- Incidental accents seen in sections: mint `#C9E0D4`, pink `#E8C9DC`, blue-white `#EAF0FF`, gray `#D8DCE2` (use sparingly for status/category, not primary).

Type:
- **Chakra Petch** — display / headings / labels (techy, geometric).
- **Inter Tight** — body / UI text.
- **JetBrains Mono** — data, metrics, codes, coded sequences (perfect for OTTO-Q stall codes / sequences).

How to apply (shadcn/Tailwind): map these into each app's `index.css` `:root` (`--background`, `--foreground`, `--primary`=red, `--card`, `--border`, `--muted-foreground`) + `tailwind.config.ts` fontFamily (sans=Inter Tight, mono=JetBrains Mono, display/heading=Chakra Petch), and load the 3 Google Fonts. Keep dark mode as the default. The OTTO-Q "Engine"/agent surfaces should lean on red + JetBrains Mono for a "command console" feel.
