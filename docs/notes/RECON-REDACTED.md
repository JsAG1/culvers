# culvers.com — architecture notes (2026-06-09)

> High-level infrastructure reconnaissance. Sensitive credentials, session data,
> and operational attack guidance have been removed from this public version.

## Stack overview

```
              Visitor (US IP required)
                       │
        ┌──────────────┼─────────────┬─────────────────────┐
        ▼              ▼             ▼                     ▼
  www.culvers.com  welcome.        orderonline.       api.culvers.com
  (CloudFront +    culvers.com     culvers.com        (Azure FD +
   Check Point     (Azure FD →     (CloudFlare →      Azure APIM)
   AppSec WAF →    Azure AD B2C)   whitelabel.olo.com)
   Azure FD →                          │
   Next.js SSR)                        ▼
   Agility CMS              Olo whitelabel Next.js
                            (serve-next.olo.com bundles)
                                       │
                                       ▼
                            api.olo.com (AWS ELB)
                            + olopayjs.s3.amazonaws.com
                              (Olo Pay = Stripe Connect)
```

## Domains and infrastructure

| Host | CDN/WAF | Origin |
|---|---|---|
| `www.culvers.com` | CloudFront + Check Point Application Security WAF | Next.js on Azure Front Door |
| `culvers.com` (apex) | CloudFront | Same as www |
| `api.culvers.com` | Azure Front Door | Azure API Management (`/gatewaygraphql`) |
| `welcome.culvers.com` | Azure Front Door | Azure AD B2C tenant |
| `orderonline.culvers.com` | CloudFlare | Olo Whitelabel Next.js |
| `cdn.culvers.com` | CloudFront | Static assets (images, logos) |
| `m.culvers.com` | — | Legacy mobile site |

DNS provider: `zaccodigitaltrustlabs.com/.net/.se`.

## Frontend — www.culvers.com

- **Next.js** SSR (`x-powered-by: Next.js`, `__NEXT_DATA__` present)
- `buildId: W2.1.1`
- **Agility CMS** (`AgilityCms.PreviewMode: false`, sitemapNode-based routing)
- Public runtime config (from `__NEXT_DATA__.runtimeConfig`):
  - GraphQL endpoint: `https://api.culvers.com/gatewaygraphql`
  - Live Olo integration enabled
  - Third-party SDKs: Radar.io (location), Amplitude (analytics)
- Edge cache: Next.js ISR (`x-nextjs-cache: STALE/HIT`)

### Key routes

```
/                                — home
/menu                            — categories + flavor-of-the-day
/menu/butterburgers              — category
/menu/butterburgers/<slug>       — product detail pages
/menu/fresh-frozen-custard       — category
/flavor-of-the-day               — featured flavor
/locator                         — restaurant locator (Radar SDK + Google Maps)
/account                         — login (redirects to Azure B2C)
/api/auth/redirect               — Next.js OAuth callback (msal.js.node)
/myculvers                       — 307 → /delicious-rewards
/rewards                         — loyalty hub
/delicious-rewards               — rewards portal
/gift-cards                      — gift cards
/orderonline                     — 404 on main site; ordering flows through locator
```

"Order Now" on product pages routes through `/locator` first. Users pick a store,
then get redirected to `orderonline.culvers.com` with store context in the query string.

## Authentication — Azure AD B2C

Opening `/account` redirects to Azure AD B2C on `welcome.culvers.com` using a custom
policy (`b2c_1a_myculvers_signuporsignin`) with PKCE and standard OAuth2 parameters.

| Parameter | Notes |
|---|---|
| Tenant | `culversb2cprodv2.onmicrosoft.com` |
| Auth library | `msal.js.node` in Next.js server-side flow |
| Redirect | `https://www.culvers.com/api/auth/redirect` (`response_mode=form_post`) |
| Federated IdPs | Google, Apple |
| OIDC discovery | Hosted on `welcome.culvers.com` with policy query parameter |

The B2C UI is Azure-hosted with branded templates. Federation buttons redirect to
external identity providers with standard OAuth state handling.

## Order and checkout — Olo Whitelabel

- **Order portal**: `orderonline.culvers.com` → CNAME `whitelabel.olo.com` → CloudFlare
- Headers include Olo correlation tokens (`olo-trace-id`, `olo-ct`)
- Frontend: Next.js bundles served from `serve-next.olo.com`
- Loading shell uses shimmer placeholders while client bundles hydrate
- Integrations observed in bundles:
  - CloudFlare Turnstile captcha
  - Google Maps in checkout
  - Mixpanel analytics
- Backend API: `api.olo.com` (AWS ELB)
- Payments: **Olo Pay** (Stripe Connect-based SDK from `olopayjs.s3.amazonaws.com`)

## Bot and WAF layers

| Layer | Domain | Behavior |
|---|---|---|
| Check Point Application Security | `www.culvers.com` via CloudFront | Blocks non-US / datacenter traffic with WAF HTML response |
| Azure API Management | `api.culvers.com/gatewaygraphql` | Requires `Ocp-Apim-Subscription-Key` header |
| CloudFlare | `orderonline.culvers.com` | Standard CF bot management (`__cf_bm` cookie) |
| CloudFlare Turnstile | Olo checkout flow | Captcha challenge in payment steps |
| Azure AD B2C | `welcome.culvers.com` | OAuth2 redirect URI validation, PKCE, CSRF state tokens |

## Research methodology

Public pages were fetched using residential US proxy routing and browser-like TLS
fingerprints (`curl_cffi` with Chrome impersonation) because the main site WAF blocks
typical datacenter IP ranges.
