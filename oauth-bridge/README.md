# oauth-bridge — cTrader OAuth redirect bridge page

A single static HTML page that makes cTrader Open API OAuth work with this mobile app.

## Why this exists

cTrader's Open API application settings **only accept an `http`/`https` redirect URI**. A custom app
scheme like `marketgrid://oauth/callback` is rejected with *"the redirect URI format is incorrect."*
But a React Native app can't be the target of a plain `https://` redirect without extra domain
verification (Android App Links).

The bridge closes that gap:

```
cTrader  ──redirect──▶  https://<your-host>/oauth/callback?code=XYZ   (this page)
                                    │  reads ?code, rebuilds the URL
                                    ▼
                        marketgrid://oauth/callback?code=XYZ           (deep link)
                                    │  the app's in-app browser is watching this scheme
                                    ▼
                        app exchanges the code for tokens
```

- The **https URL of this page** is what you register with cTrader and enter as the *Redirect URI* on
  the app's login screen. It is the `redirect_uri` sent on both authorize and token exchange.
- `marketgrid://oauth/callback` is only what the browser watches and what this page bounces to. It is
  never sent to cTrader. It must stay in sync with `OAUTH_DEEPLINK_RETURN_URL` in
  [`market-grid-mobile/src/adapters/ctrader/oauth-token.ts`](../market-grid-mobile/src/adapters/ctrader/oauth-token.ts)
  and the `scheme` in `market-grid-mobile/app.config.ts` (`marketgrid`).

No secrets touch this page — only the short-lived one-time authorization code is relayed. The token
exchange (which needs the client secret) happens on the device.

## Deploy it

Host `index.html` at a stable https URL on any static host — no server code required:

- **GitHub Pages** — commit `index.html` to a repo, enable Pages; URL is
  `https://<user>.github.io/<repo>/`.
- **Cloudflare Pages / Netlify / Vercel** — drag-and-drop or connect this folder; each gives an https URL.
- **Your own domain** — upload `index.html` to any static host.

The page works whether it's served at `/` (as `index.html`) or at a subpath — the redirect URI you
register just has to point at wherever it lands (e.g. `https://you.github.io/mg-oauth/` or
`https://you.dev/oauth/callback`).

## Register it (two places must match exactly)

1. **cTrader** — [connect.spotware.com/apps](https://connect.spotware.com/apps) → your application →
   *Redirect URIs* → add the exact https URL where you deployed this page.
2. **The app** — on the login screen, set *Redirect URI* to the **same** https URL (or import it via the
   credentials JSON / `EXPO_PUBLIC_CTRADER_REDIRECT_URI`).

The two values must be byte-identical (trailing slash included) or cTrader rejects the token exchange.

## Test it

Open `https://<your-host>/oauth/callback?code=test123` in a mobile browser with the dev build installed —
it should bounce to `marketgrid://oauth/callback?code=test123` and open the app. Add `?error=access_denied`
instead to see the failure state.
