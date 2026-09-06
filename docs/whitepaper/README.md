# Whitepapers & customer-facing assurance documents

The public home for VEV's downloadable, customer-facing security and assurance
documents. These are written at **customer altitude** (evidence a CISO can read),
never internal control specifics, exploit material or private incident detail.

## Documents

| Document | Source | Download |
| --- | --- | --- |
| Atlas Security Whitepaper (v1.0) | [`atlas-security-whitepaper.html`](./atlas-security-whitepaper.html) | [`atlas-security-whitepaper.pdf`](./atlas-security-whitepaper.pdf) |

## How these are built (reusable template)

Each whitepaper is a **single, self-contained HTML file** styled to the VEV design
system, rendered to PDF with headless Chromium. This keeps brand fidelity (Space
Grotesk, paper/ink palette, the woven glyph, inline-SVG diagrams) and makes the
source auditable and diffable.

1. Author the content in the HTML file. Palette and type mirror `tokens.css` from
   the Homepage design system; the brand font is vendored under `assets/`
   (`space-grotesk-latin.woff2`, SIL OFL).
2. Serve the folder over HTTP so the font loads:
   ```bash
   python -m http.server 8792
   ```
3. Render to PDF (A4, backgrounds, page numbers). Any headless-Chromium PDF path
   works; for example the gstack browse tool:
   ```bash
   browse goto http://localhost:8792/atlas-security-whitepaper.html
   browse pdf atlas-security-whitepaper.pdf --format a4 --print-background --page-numbers
   ```
4. Commit both the HTML source and the built PDF.

New assurance documents (DPA summary, product one-pagers, further whitepapers)
should reuse this pattern and the shared cover/section/diagram styling so the set
reads as one system.

## Editorial rules

- **Public-safe.** Customer altitude only. No internal control specifics, exploit
  material, private incident detail, internal hostnames, or entitlement/licence
  enforcement internals.
- **Honest tense.** Present tense only for controls that are live; everything else
  is tagged *On the way*. Never list a certificate we do not hold.
- **Versioned.** Each document carries a version and date on the cover and is
  updated as controls land.
