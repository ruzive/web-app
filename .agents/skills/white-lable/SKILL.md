---
name: white-label
description: Use this whenever the user wants to white-label, rebrand, or theme an open-source project with their own company's logo, colors, name, or visual identity — including requests like "customize this to our theme", "swap the branding", "make this look like our product", or "rebrand this app". Applies to any open-source codebase; includes extra guidance for Angular-based apps like the Mifos web app.
---

# White-Label / Rebranding Skill

Goal: apply a company's visual identity to an open-source project **without**
touching business logic, and in a way that survives pulling future upstream
updates with minimal merge conflicts.

## Step 1: Inventory the branding surface

Before changing anything, find every place branding appears. Search the repo
for these categories rather than guessing:

- **Logo/images**: `assets/`, `public/`, `static/`, `images/`, `img/`
  directories — logo files, favicons, splash screens, app icons
  (`manifest.json`, `apple-touch-icon`, PWA icons at multiple resolutions).
- **Colors/theme**: SCSS/CSS variables, theme config files (look for
  `_theme.scss`, `theme.ts`, `tailwind.config.js`, CSS custom properties like
  `--primary-color`), any centralized design-token file.
- **App name/title**: `index.html` `<title>`, `package.json` `name`,
  `manifest.json` `name`/`short_name`, login/splash screen text, email
  templates, footer/about pages.
- **Config-driven branding**: check `environment.ts` / `.env` / config files
  first — some apps already expose logo path, primary color, or app name as
  config rather than hardcoded, which is the easiest case.

Report this inventory back before making changes, so scope is agreed before
editing.

## Step 2: Isolate the changes

To keep this mergeable against upstream updates:

- Prefer **overriding config values** over editing source files directly,
  wherever the app supports it.
- If theme is SCSS variables, override them in a single dedicated file
  (e.g. `_custom-theme.scss`) imported last, rather than editing the
  upstream theme file in place.
- Replace asset files in-place only for files that are _pure assets_
  (logos, icons) — that's low-risk for merge conflicts since binary/image
  files don't diff line-by-line.
- Avoid renaming files or restructuring folders unless necessary — that
  maximizes future merge pain.

## Step 3: Apply changes

1. Swap logo/icon/favicon files (keep original filenames and dimensions
   unless the app's code references them by name — check first).
2. Update the theme/color variables to the corporate palette.
3. Update app name/title strings across `index.html`, `manifest.json`,
   `package.json`, and any hardcoded UI text.
4. Update login/splash screens and footer/about text if present.

### If a required asset variant is missing

Apps typically need several renderings of the same logo (full lockup,
icon-only, favicon, apple-touch-icon, light-bg variant, dark-bg variant).
Before treating a missing variant as something to source or design from
scratch, check whether it can be **derived** from an existing master file:

1. If the master logo has a flattened/baked-in background (common with
   AI-generated or exported marketing logos), isolate the mark first —
   chroma-key it out if the background is a near-uniform color, rather
   than assuming a transparent source exists.
2. Derive size/format variants mechanically: resize, square-crop for
   icon slots, build a multi-resolution `.ico` for favicons.
3. **Before producing a light-background and dark-background variant from
   one source, check the actual pixel luminance of the opaque logo
   pixels.** Metallic/chrome/gradient logos often have a wide luminance
   spread (bright highlights + dark shadows) — meaning a flat transparent
   version will have real contrast dropout on one background or the
   other (highlights vanish on white, shadows vanish on dark). Don't
   silently ship a variant with dropout. If dropout is likely, place the
   mark on a small solid/rounded chip in the brand's dark color instead
   of forcing transparency — and say so explicitly, since a true flat
   single-tone version is a design decision, not a technical derivation.
4. Flag fine detail that won't hold up at small sizes (favicons at
   16×16 in particular) rather than shipping a blurry icon silently.

## Step 4: Verify

- Run the app locally and check: login page, main nav/header, favicon in
  browser tab, PWA install icon (if applicable), any email templates or
  PDF/report headers that also carry the old logo.
- Check responsive/mobile views — logos sized for desktop nav often break
  on mobile.
- Check dark mode / alternate themes if the app has them — a swapped
  logo may not have proper contrast in both.

## Mifos Web App specifics (Angular)

Verified against the real `openMF/web-app` repo:

- **Logo — config-driven, no code edit needed.** `environment.ts` reads
  `tenantLogoUrl` / `tenantLogoUrlDark` from a `TENANT_LOGO_URL` env var
  (documented in `env.sample`). The login component (`login.component.ts`)
  consumes these directly, with fallback file paths
  `assets/images/default_home.png` (light) and
  `assets/images/white-mifos.png` (dark) if unset. Setting the env var at
  deploy time is enough for the primary logo — only fall back to
  overwriting the asset files if you need it baked in at build time.
- **Other logo/icon files** live in `src/assets/images/`:
  `MifosX_logo.png` (nav), `MifosX_logoSmall.png` (collapsed sidebar icon),
  plus `favicon.ico` and `apple-touch-icon.png` at repo root of `src/` —
  these are hardcoded in `src/index.html` with no env override, so they
  need direct file replacement (see the derivation step above for
  generating them all from one master logo).
- **App name is an i18n key, not hardcoded** — `APP_NAME` is used in the
  page title, login hero, sidenav, and footer, but it's duplicated across
  **13 separate locale files** under `src/assets/translations/`
  (`en-US.json`, `es-MX.json`, `fr-FR.json`, etc.). Renaming the app means
  editing the key in all 13 files, not one.
- **Colors** are centralized in `src/theme/_material-palette.scss` as
  `$primary-palette` / `$accent-palette` SCSS maps for light and dark
  mode — single-file override, don't touch the individual
  `*.component-theme.scss` files that consume it.
- **Watch out**: `src/theme/` also contains five unrelated pre-built
  Material theme files (`indigo-pink.scss`, `deeppurple-amber.scss`,
  `pictonblue-yellowgreen.scss`, `pink-bluegrey.scss`,
  `purple-green.scss`) — don't edit these; the app uses
  `mifosx-theme.scss`, which imports `_material-palette.scss`.
- After rebuilding (`ng build`), check the browser tab favicon and PWA
  install prompt separately from the in-app logo — `manifest.json` icons
  are referenced independently and are easy to miss.
