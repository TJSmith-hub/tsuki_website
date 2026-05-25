# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Marketing site for Tsuki Mel Tattoo (Melissa Hall), a tattoo artist in Leith, Edinburgh. It is a **single-page Astro site** (one route: `src/pages/index.astro`) deployed to **Cloudflare Workers** via the `@astrojs/cloudflare` adapter, with all navigation handled by in-page anchors (`#about`, `#black-grey`, `#colour`, `#flash`, `#booking`).

## Commands

```bash
npm run dev              # astro dev — local dev server
npm run build            # astro build — outputs to ./dist
npm run preview          # astro preview — preview the built site
npm run generate-types   # wrangler types — regenerate worker-configuration.d.ts
```

There are no tests, lint, or typecheck scripts configured. `npx astro check` requires `@astrojs/check` to be installed on first run.

## Architecture

### Page composition
`src/pages/index.astro` is the only page. It wires `<Layout>` around `<NavBar> + <Hero> + <About> + 3× <GallerySection> + <BookingCTA> + <Footer>`. Each component is a self-contained section with its own background and `<InkDivider>` SVG transitions between adjacent sections.

### Gallery is convention-driven
The three `<GallerySection>` instances are populated from `src/assets/gallery/{black-grey,colour,flash}/` via `import.meta.glob(..., { eager: true })` in `src/pages/index.astro`. To add a tattoo to a gallery, drop the image file into the matching folder — no code change needed. Files are sorted alphabetically; the alt text is auto-derived from the filename (hyphens/underscores become spaces). Use descriptive filenames because they become SEO-visible alt text.

### Shared constants and icon
- `src/constants.ts` — `BOOKING_URL` (Google Form) and `INSTAGRAM_URL`. Imported by NavBar, Hero, BookingCTA, Footer. Change the URL here, not in components.
- `src/components/InstagramIcon.astro` — the only place the Instagram SVG `<path>` lives. Accepts a `class` prop for sizing.
- The Footer's "Book" link intentionally uses a local `BOOKING_ANCHOR = '#booking'` (in-page scroll to the `<BookingCTA>` section) rather than the external `BOOKING_URL`. Don't "fix" this without confirming.

### Styling: Tailwind v4 + theme tokens in CSS
This project uses **Tailwind v4** via the `@tailwindcss/vite` Vite plugin (configured in `astro.config.mjs`), **not** the `@astrojs/tailwind` integration and **not** a `tailwind.config.*` file. The design tokens are declared inside `src/styles/global.css` using Tailwind v4's `@theme` directive:

- Colours: `aubergine`, `aubergine-light`, `plum`, `lavender`, `lavender-dark`, `rose-dust`, `cream` — used as `bg-aubergine`, `text-lavender`, etc.
- Fonts: `font-script` (Allura), `font-heading` (Cormorant Garamond), `font-body` (Raleway), loaded from Google Fonts in `Layout.astro`.

To add or change a token, edit `src/styles/global.css`. Don't add a `tailwind.config.js`.

### SEO and structured data
`src/layouts/Layout.astro` renders a per-page Schema.org `@graph` (a `Person` for Melissa Hall and a `TattooParlor` for the studio, linked via `@id`) as inline JSON-LD. The `<` chars are escaped to `<` to prevent script-tag breakout. Title/description default to artist-focused values but can be overridden per-page via props. The layout also emits OpenGraph, Twitter Card, `robots`/`googlebot` AI opt-out meta, geo meta, and the canonical URL.

### CSP — important constraint
`public/_headers` sets a Content-Security-Policy that includes `script-src 'self' 'unsafe-inline' https://static.cloudflareinsights.com`. The `'unsafe-inline'` is required because the JSON-LD payload varies per page (dynamic title/description), so it can't be hashed at build time. The trade-off is documented in the file. Before adding any other inline `<script>`, consider whether it should be external instead — the current `'unsafe-inline'` exists for JSON-LD only.

`public/_headers` also defines long-cache rules for `/_astro/*` (immutable, 1 year) and `/gallery/*`, `/images/*` (7 days).

### Cloudflare deployment
- Adapter: `@astrojs/cloudflare` with `imageService: 'compile'` (build-time image optimization). Source images can be large; the adapter generates optimized variants.
- `wrangler.jsonc` — worker name `tsuki-website`, serves `./dist` as static assets via the `ASSETS` binding. `observability.enabled: true`.
- Site origin is `https://tsuki.tattoo` (set in `astro.config.mjs` and used by the sitemap integration).
