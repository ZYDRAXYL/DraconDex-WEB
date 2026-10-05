<p align="center">
  <img src="assets/brand/DraconDex_Color-512.png" alt="DraconDex logo" width="140">
</p>

<h1 align="center">DraconDex-WEB</h1>

<p align="center">
  The website for <a href="https://github.com/ZYDRAXYL/DraconDex-APP">DraconDex</a>,
  deployed to GitHub Pages.
</p>

<p align="center">
  <strong><a href="https://zydraxyl.github.io/DraconDex-WEB/">zydraxyl.github.io/DraconDex-WEB</a></strong>
</p>

---

## What's here

Three static pages, no build step, no dependencies:

| Page | Covers |
|---|---|
| `index.html` | What DraconDex is, its features, the v3 module tree, a plugin teaser, and links to the docs |
| `download.html` | Every way to get the app in one place — the latest release's Windows assets and the browser/PWA build side by side, which build to pick, checksum verification, and the signed Android APKs |
| `plugins.html` | Every official plugin, how installing from a link works, what the sandbox does and doesn't allow, and how to write your own |

Supporting files:

```
assets/css/site.css     one stylesheet; colors and fonts mirror the app's design tokens
assets/js/icons.js      shared inline-SVG icon set (Lucide paths), used in place of emoji
assets/js/theme.js      midnight/daylight/moonlight theme switch, persisted to localStorage
assets/js/lang.js       English/Thai language switch, persisted to localStorage
assets/js/strings.th.js the Thai dictionary that assets/js/lang.js swaps in
assets/js/releases.js   reads assets/data/releases.json for the download pages,
                        splitting it into the desktop (`v*`) and Android
                        (`flutter-v*`) release streams
assets/data/releases.json  the release list itself — a static snapshot of this
                        repo's own GitHub Releases, refreshed by the app repo's
                        CI after every publish (see below), not hand-edited
assets/js/plugins.js    the plugin catalogue, refreshed from live manifests
assets/brand/           logo and icon, downscaled from the app repository
assets/fonts/           self-hosted Kanit + IBM Plex Sans Thai, Latin+Thai subsets only
assets/screenshots/     app screenshots for the module-tree and theme-mockup sections on index.html
.nojekyll               serve the files as-is, no Jekyll processing
```

## Theme switch and language switch

The topbar has two buttons next to each other: a language toggle (`TH`/`EN`)
and a theme toggle. Both mirror the same pattern — a small script loaded
synchronously in `<head>` so the stored choice (`localStorage`) applies before
first paint, avoiding a flash of the wrong theme or language.

- **Theme** cycles through the app's three built-in themes — midnight
  (default, dark), daylight (light) and moonlight (dark blue) — matching
  `src/design/tokens/tokens.json` in the app repository. With nothing stored,
  it follows the OS light/dark preference (between midnight and daylight
  only; moonlight is always an explicit pick).
- **Language** swaps every element tagged `data-i18n="key"` (and
  `data-i18n-attr="attr:key"` for attributes like `aria-label` or `alt`)
  against the Thai dictionary in `assets/js/strings.th.js`. English lives
  directly in the HTML — nothing needs to be listed there for the English
  side. Content injected at runtime by `releases.js`/`plugins.js` (release
  labels, plugin cards) is sourced live from GitHub and is not translated.

The index page also has a "Theme mockups" section using the same coverflow
carousel as the module-tree screenshots (`assets/js/mod-slider.js` already
initializes every `[data-mod-slider]` element on the page, so a second
carousel instance needs no extra JS) to show off daylight, moonlight and a
few of the example palettes from the app's in-app Custom Theme editor.

## Two release streams, one tag list

The app repository publishes two independent release lines that both land in
this repo's own release list:

| Tags | Built by | Assets |
|---|---|---|
| `v<x.y.z>` | `build-electron.yml` | the three Windows builds + `checksums-sha256.txt` |
| `flutter-v<x.y.z>` | `build-apk.yml` | four APKs (three ABIs + universal) + `checksums-sha256.txt` |

Their version numbers are unrelated, and the newest release *overall* is
routinely the Android one, so `assets/js/releases.js` filters by stream before
anything else — the Windows download button, the latest-assets list and the
release history all read the desktop stream, and the APK section on
`download.html` reads the Android one. Dropping that filter puts `.apk` files
under the Windows download button the next time an APK ships last.

## Why the release data is a committed file, not a live API call

Release versions and asset sizes change in *another* repository
(`ZYDRAXYL/DraconDex-APP`, which is private). Rather than calling that other
repo's API from every visitor's browser, `assets/js/releases.js` reads a file
committed straight into **this** repo:

- **`assets/data/releases.json`** is a snapshot of this repo's own GitHub
  Releases — the app repo's build workflows publish each release, notes and
  assets alike, directly onto `ZYDRAXYL/DraconDex-WEB` (this repo) as a normal
  GitHub Release, then `.github/scripts/update-web-releases-json.sh` (in the
  app repo) re-reads that list via the Contents API and commits it here as
  this file. Its shape is the raw `GET /repos/.../releases` response,
  unmodified, so `releases.js`'s parsing needed no changes when the source
  moved from a live fetch to a static one — only the fetch target did.
  Reading a same-origin static file this way means no token, no CORS
  question, and no rate limit: the old approach called
  `api.github.com/repos/ZYDRAXYL/DraconDex-REL/releases` from the browser,
  which is capped at 60 anonymous requests/hour **per IP** — fine for one
  visitor, but a shared office/campus NAT or a traffic spike could exhaust it
  for everyone behind that IP at once. `ZYDRAXYL/DraconDex-REL` still exists
  as a separate public mirror; only the in-app update check
  (`electron/src/db/update.js`, `flutter/lib/data/services/update_service.dart`
  in the app repo) reads it now.
  Nothing fills this file from here: it is written by the app repo's
  "Mirror releases" workflow, which needs a `WEB_REPO_TOKEN` secret there
  holding a token with `contents: write` on this repo. Until that secret
  exists and the workflow has run once with `tag: all` (its backfill lever),
  this repo has no Releases, the file stays `[]`, and every download entry
  point falls back to its "no releases published yet" state.
  Responses are still cached in `sessionStorage` for ten minutes — cheap
  insurance against re-fetching the same static file on every page within a
  session.
  The list is sorted by version number rather than taken in the order the
  file lists them: that order follows each release's `created_at`, which is
  the date of the *commit* its tag points at, so a tag cut on an older commit
  sorts below releases published days earlier.
- **Plugin manifests** still come from `raw.githubusercontent.com`, which is
  CORS-open and outside any GitHub API rate limit. Each card ships with the
  manifest values baked in, so a failed fetch is a no-op rather than an empty
  page.

## Local preview

Any static file server works — the pages use relative links only:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

## Deployment

`.github/workflows/deploy-pages.yml` publishes the repository root to GitHub
Pages on every push to `main`, and can also be run manually from the Actions
tab. It needs **Settings → Pages → Source** set to **GitHub Actions** once.

## Keeping it in sync with the app

A few things here are copies of facts that live in
[`ZYDRAXYL/DraconDex-APP`](https://github.com/ZYDRAXYL/DraconDex-APP) and need
updating when that repository changes:

- The color tokens at the top of `assets/css/site.css` (and the theme cycle
  in `assets/js/theme.js`) mirror `src/design/tokens/tokens.json` — currently
  three themes: midnight, daylight, moonlight.
- The `theme-*.png` files in `assets/screenshots/` (see the README there) are
  exported from the same `docs/mockups/` as the module-tree screenshots.
- The release asset names matched in `assets/js/releases.js` come from
  `.github/workflows/build-electron.yml` (desktop) and
  `.github/workflows/build-apk.yml` (Android).
- The Android signing certificate quoted in `download.html`'s `#android`
  callout — subject `CN=DraconDex, O=ZYDRAXYL, C=TH`, SHA-256 `6b1b62e9…c2dc97e8`,
  and the "2.3.0–2.8.0 must uninstall first" caveat — comes from
  `docs/UPDATE.md` §2.16.
- The browser build behind `#web` — a block inside `download.html`'s Download
  section rather than a section of its own — is a third repository,
  [`ZYDRAXYL/DraconDex-PWA`](https://github.com/ZYDRAXYL/DraconDex-PWA), which builds
  both front-ends for the web and deploys them to
  [zydraxyl.github.io/DraconDex-PWA](https://zydraxyl.github.io/DraconDex-PWA/). What
  that page says a browser cannot do comes from its README and
  `docs/BROWSER-BUILD.md`.
- The manifest limits listed on `plugins.html` come from `docs/PLUGINS.md`.
- The logo files in `assets/brand/` come from `src/assets/brand/`. The master
  `DraconDex_Color.png` is 1839px and ~550 KB, so the site ships 512px and
  64px copies of it instead — regenerate both if the mark ever changes.

To add a plugin to the catalogue, append an entry to the `PLUGINS` array in
`assets/js/plugins.js` — both the home page teaser and the plugins page read
from it.

## License

See [LICENSE](LICENSE) and [NOTICE](NOTICE). Created by LDKTC.
