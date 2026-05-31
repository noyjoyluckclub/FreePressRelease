# FreePressRelease — site/app integration

Two surfaces make up the product:

| Surface | Host | Owns |
| --- | --- | --- |
| **Static marketing site** | `freepressrelease.io` (this folder) | Landing, policies, contact, and the **public release pages** at `/press/<slug>` |
| **Portal app** | `my.freepressrelease.io` (`pressroom/apps/freepressrelease`) | Auth (login/signup/forgot/verify), dashboard, submit, admin, API, and the `/press/<slug>` **renderer** |

The release pages are *rendered* by the portal app (they're DB-backed) but *served*
under the apex so the public URL is `freepressrelease.io/press/<slug>`.

## 1. `/press` reverse proxy (this site)

The apex `/press/*` is proxied to the portal app so the public URL is preserved:

```
freepressrelease.io/press/my-title  →  my.freepressrelease.io/press/my-title
```

**This site is hosted on Netlify, so `_redirects` is the active config.** Netlify
status `200` rules are rewrites/proxies (the browser URL stays on the apex); a
`301/302` would change the visible URL to `my.freepressrelease.io` and defeat the
apex-canonical goal. The `_redirects` file must be in the Netlify **publish
directory** (the repo root here, since this is a no-build static site) — confirm
your site's Publish directory in Netlify is `.` / root, or move `_redirects`
accordingly. After deploy, verify at `https://app.netlify.com` → Deploys → the
latest deploy shows the redirect rules were processed.

`.htaccess` (also in this repo) is the equivalent for Apache/cPanel hosts and is
ignored by Netlify — keep it only as a fallback if the site ever moves hosts.

Both configs also proxy `/press-sitemap.xml` and `/press-rss.xml` to the app's
`/sitemap.xml` and `/rss` (DB-backed). Reference `/press-sitemap.xml` from this
site's `robots.txt`/sitemap so the release URLs get indexed under the apex.

## 2. Canonical URLs (portal app)

The app builds public release URLs from `PUBLIC_SITE_URL` (default
`https://freepressrelease.io`):

- `app/press/[slug]/page.tsx` sets `<link rel="canonical">` and `og:url` to
  `PUBLIC_SITE_URL/press/<slug>`.
- `sitemap.ts`, `rss/route.ts`, submission approval emails, and the dashboard
  "view public page" link all use the same apex `/press/<slug>` form.
- `robots.ts` on `my.freepressrelease.io` disallows all crawling (the portal is
  private); indexing happens on the apex.

Set `PUBLIC_SITE_URL=https://freepressrelease.io` in the app's environment
(`.env.local` / hosting env). It defaults to the apex if unset.

## 3. Auth handoff (this site)

The static auth pages are gateways — they do **not** authenticate. The header
**Login** CTA links straight to `my.freepressrelease.io/login`, and the
login/signup/forgot forms hand off to the matching portal route (forwarding only
the typed email as a prefill hint; credentials are never posted from the static
site). The portal app owns the real, secure auth flow.

## Open item

There is no public **index/listing** of all releases on the apex (the app's old
`/latest` and `/category` browse pages were removed when the portal was slimmed to
auth + dashboard). If release discovery is needed on `freepressrelease.io`, add a
`/press` (or `/latest`) listing — either a static page that links to the proxied
app listing, or restore a read-only listing route in the app behind the proxy.
