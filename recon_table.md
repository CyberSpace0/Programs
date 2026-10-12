# PlayStation Recon Table (from `httpx` output)

Source: `httpx` (239 live hosts) against `domains` / `subs`.
Output format in `httpx` file: `URL [redirect] [title/status] [server] [tech]`.

## 0. Overview

| Pattern in httpx output | Count | Meaning for recon |
|---|---|---|
| `Access Denied` + `AkamaiGHost` | 176 | Akamai WAF blocks the request. Host exists, but content is not visible. Test with care and check the program's rules on bypass testing. |
| `Akamai Bot Manager` | 29 | Bot protection. Expect 403/503 and JS challenges. |
| `SAP Commerce Cloud - Forbidden` | 10 | Hybris/SAP Commerce backends (`commerce-api.*`). High-value, see section 3. |
| `Next.js` / `React` / `Webpack` | 2 | Frontend SPA. Check JS bundles, source maps, and env vars. |
| `Java` | 2 | Java backend (`entitlements.api`, `help`). Check Spring/Actuator-style endpoints. |
| `Apache` / `nginx` with origin-looking title | 14 | Possibly non-Akamai or direct origins. Good candidates for origin-IP checks (only if in scope). |
| `HTTP 406 Not Acceptable` | 4 | Content negotiation. Retry with `Accept: application/json` and `Content-Type` headers. |

Environment tags in hostnames (inferred, verify against docs): `d1-*`, `q1-*`, `e1-*`, `sp-int`, `pqa`, `pmgt`, `np`, `mgmt`, `prod-qa`, `spint`, `idmapper`. These look like dev, QA, staging, integration, and management tiers. Non-prod hosts often have weaker auth or debug features, but **confirm they are in scope first**.

## 1. Notable hosts (prioritized)

| Subdomain | Status / title | Server / tech | What it likely services | Expected tech | Possible vulnerability areas to test |
|---|---|---|---|---|---|
| `signin.playstation.net` | nginx, home bookmark redirect | nginx, HSTS | Sign-in / Okta-based SSO portal (OAuth2 `default` auth server) | Okta, OIDC/OAuth2 | Open redirect in `redirect_uri`/`returnUrl`, OAuth state/PKCE issues, account enumeration |
| `help.playstation.net` | Redirects to `signin.playstation.net/oauth2/.../authorize` with `redirect_uri=help.playstation.net/navpage.do` | `snow_adc`, Java | Help center (ServiceNow-style `navpage.do`) | ServiceNow, Java | OAuth `redirect_uri` manipulation, `navpage.do` parameter injection, reflected XSS, SSRF through help links |
| `docs.playstation.net` | Redirect to `/unsupported-browser?origin=` | Apache, Amazon ALB, AWS | Developer documentation | Apache, AWS ALB | Reflected XSS / open redirect through `origin=` parameter |
| `dashboard.playstation.net` | 301 to `/sb/<id>` | nginx, Akamai | Developer dashboard (sandbox/SB) | nginx | IDOR on `/sb/<id>`, auth bypass, access control on developer objects |
| `partners.playstation.net` | "PlayStation Partners" | nginx, Akamai, HSTS | Partner portal (publisher/dev hub) | nginx | Auth/IDOR on partner resources, file upload, business logic |
| `develop.playstation.net` | Redirect to `partners.../hub` | Akamai | Developer entry point | Akamai | Redirect chain testing, legacy path exposure |
| `publish.playstation.net` | "Sony Content Pipeline" | None shown | Game/content publishing pipeline | Unknown (CMS/pipeline) | Upload/storage exposure, auth on upload endpoints, IDOR on content IDs |
| `register.playstation.net` | "PlayStation Partner Registration" | Akamai | Partner registration | Akamai | Mass assignment, enumeration, weak verification flows |
| `learn.playstation.net` | Redirect to `/auth/login/?redirect=%2F` | Akamai, HSTS | Training/learning portal | Unknown | Open redirect via `redirect=`, login CSRF |
| `session-share.playstation.com` | 404 | Next.js, Node.js, React, Webpack | Session sharing (front-end) | Next.js, Node.js, React | Next.js middleware auth bypass (check version), exposed `_next/static` source maps, leaked secrets in JS |
| `profile.playstation.com` | No title | Next.js, React, Webpack | User profile SPA | Next.js, React | IDOR on profile APIs, JS secret leakage, source maps, XSS via profile fields |
| `store.playstation.com` | Redirects to `/en-us/` | Akamai, HSTS | PlayStation Store (web) | Akamai | Store logic, price/coupon tampering on backend APIs, locale handling |
| `checkout.playstation.com` | Redirects to `/cart` | Akamai | Checkout flow | Akamai | Price manipulation, step bypass, coupon abuse, race conditions |
| `transact.playstation.com` | "PlayStation Store Transact" | AkamaiNetStorage | Transaction / purchase handling | Akamai | Transaction tampering, IDOR on orders, payment callback validation |
| `wallets.api.playstation.com` | 429 Too Many Requests | nginx, Akamai | Wallet / balance API | nginx | Rate-limit bypass, IDOR on wallet IDs (if allowed to test) |
| `entitlements.api.playstation.com` | No title | Java, Akamai, HSTS | Entitlement / license checks for content | Java | Entitlement bypass, IDOR, Java deserialization, Spring Actuator exposure |
| `core-entitlements.*` (many) | Access Denied | Akamai | Entitlements core backend | Unknown | Verify internal exposure, env-specific auth gaps |
| `cart.api.playstation.com` | No title | Akamai, HSTS | Cart API | Akamai | Price tampering, negative quantity, IDOR on cart IDs |
| `catalog.api.playstation.com` | 403 Forbidden | Apache | Product catalog API | Apache | Path/ACL bypass, `/..;/` tricks, exposed API docs |
| `gamelist.api.playstation.com` | 403 Forbidden | Apache | Game list API | Apache | Same as catalog, plus IDOR on game/user IDs |
| `api.bridges.playstation.com` | No title | Apache 2.4.68 (Amazon Linux), OpenSSL 3.5.5 | "Bridges" backend API (AWS origin-looking) | Apache on Amazon Linux | Check `/server-status`, `/server-info`, default paths, origin exposure |
| `api.ivaldi.playstation.com` | 404 Not Found | Apache | Ivaldi service (internal name) | Apache | Enumerate paths, check if API docs exposed |
| `analytics.playstation.net` | "PlayStation Partners Analytics" | AkamaiNetStorage, HSTS | Partner analytics dashboard | S3-like storage (AkamaiNetStorage) | Exposed bucket/object paths, auth on analytics endpoints, data leakage |
| `social.playstation.com` | No title | AkamaiNetStorage | Social/community features | AkamaiNetStorage | Storage misconfig, exposed assets |
| `commerce-api.direct*.playstation.com` (10 hosts) | `SAP Commerce Cloud - Forbidden` | `*`, HSTS | SAP Hybris commerce backends (direct store, multiple envs: e1-np, e1-pqa, e1-spint, e1-pmgt, e2-np, e3-np, stage, stage2, stage3, p1-np) | SAP Commerce / Hybris | `/hac` (Hybris admin console) exposure, `/occ/v2` and `/rest/v2` OCC API endpoints with anonymous access, default creds, OCC OAuth client misconfig |
| `api.direct.playstation.com` / `api.direct.e1-*.playstation.com` | 404 / Access Denied | Akamai, HSTS | API gateway for direct store | Akamai | Enumerate path bases, compare with `commerce-api.direct*` |
| `direct.playstation.com` | "Country/Language select" | AkamaiNetStorage | Regional store landing | Akamai | Locale parameter injection, open redirect |
| `ca.account.sony.com`, `my.account.sony.com` | Access Denied / no title | Akamai, AkamaiNetStorage | Sony account management | Akamai | Account takeover paths, IDOR on profile/security settings, email change flows |
| `account.sonyentertainmentnetwork.com` | Redirects to `id.sonyentertainmentnetwork.com/id/management/` | Akamai | Legacy account domain | Akamai | Legacy domain auth gaps, redirect abuse |
| `id.sonyentertainmentnetwork.com` | No title | AkamaiNetStorage, Akamai Bot Manager | Sony ID login | Akamai | Login brute force (if in scope), OAuth/OIDC issues, password reset flows |
| `ps5.np.playstation.net` | Access Denied | AkamaiGHost, HSTS | PSN NP (Network Platform) for PS5 | Akamai | NP ticket/auth endpoints, token handling |
| `web.np.playstation.com` | Access Denied | AkamaiGHost, HSTS | Web NP backend | Akamai | Same as ps5.np |
| `activate.playstation.net` | Access Denied | AkamaiGHost, HSTS | Device/product activation | Akamai | Activation code brute force, enumeration |
| `codes.playstation.net` | Redirect to https | AkamaiGHost | Redeem codes portal | Akamai | Code enumeration, rate limits, race conditions on redemption |
| `manuals.playstation.net` | Access Denied | AkamaiGHost | Product manuals | Akamai | Path traversal, exposed files |
| `subs.playstation.net` | Access Denied | AkamaiGHost | Subscriptions (PS Plus?) | Akamai | Subscription logic, IDOR |
| `announcements.playstation.net` | Service Unavailable | AkamaiGHost | Announcements feed | Akamai | Content injection, cache poisoning |
| `segments.playstation.net` | "Playstation Partners" | Akamai | Audience segments | Unknown | Data exposure |
| `unity.playstation.net` | Access Denied (http) | AkamaiGHost | Unity partner portal | Akamai | Downgrade http path, auth |
| `compete.playstation.com` | Redirect to `/tournaments` | AkamaiGHost, HSTS | Tournaments | Akamai | Redirect abuse, IDOR on tournaments |
| `communities.api.playstation.com` | Service Unavailable | AkamaiGHost | Communities API | Akamai | Auth/IDOR on community objects |
| `critic.api.playstation.com`, `critic.sp-int`, `critic.e1-np` | Access Denied / no title | Akamai | Critic / reviews service | Akamai | Review injection, IDOR, rate-limit |
| `gr-content.api.playstation.com`, `gr-reporting.api.playstation.com` | No title | Akamai | Content/reporting | Akamai | Reporting data exposure |
| `ems-delivery.api.playstation.com` | No title | Akamai | Delivery service | Akamai | Auth gaps on delivery endpoints |
| `ems-admin.e1-pmgt.api.playstation.com` | Access Denied | AkamaiGHost | EMS admin (management tier) | Akamai | Admin panel exposure, auth bypass (if in scope) |
| `dev.api.wrapup.playstation.com` | "Invalid URL" | AkamaiGHost | Dev wrapup service | Akamai | Dev environment exposure, path handling |
| `dev.api.bridges.playstation.com` | Access Denied | AkamaiGHost, HSTS | Dev bridges API | Akamai | Dev env, weaker auth |
| `accounts.api.playstation.com`, `activity.api.playstation.com`, `closure.api.playstation.com`, `customermessagecenter.api.playstation.com` | 406 / 404 | nginx, Akamai | Account and activity API family | nginx | Send `Accept: application/json`, enumerate API paths, check verbose errors |
| `bumblebee.api.playstation.com`, `facts.api.playstation.com` | Service Unavailable | Akamai | Internal-looking services | Akamai | Endpoint discovery |
| `ad-conf-svc.*`, `ads-poc.*`, `audience-segments-module.*`, `cba.*`, `capone.*`, `cis.*`, `cosmos.*`, `dms.*`, `drm.*`, `dss.*`, `dora.*`, `eliot-ness.*`, `ets.*`, `feature.*`, `garbo.*`, `genome.*`, `gms.*`, `gr-processor.*`, `helpsystem-proxy.*`, `consent-admin.*` | Access Denied (mostly) | AkamaiGHost | Internal service names across env tiers | Akamai | Good for wordlist of internal API names; test each env tier separately |

## 2. Non-prod tiers (group summary)

| Env tag | Count of Access Denied hosts | Test idea |
|---|---|---|
| `e1-np` | 30 | Non-prod API surface, weaker auth |
| `sp-int` | 16 | Integration env, may expose debug |
| `mgmt` | 14 | Management plane, admin APIs (high impact if reachable) |
| `e1-pmgt` | 12 | Pre-prod management |
| `d1-np`, `q1-pmgt`, `q1-np`, `e1-spint`, `prod-qa`, `q1-spint` | 8–10 each | Check same service across tiers for differing auth |

## 3. Recon checklist before pentest

1. **Scope**: confirm each tier (`np`, `pqa`, `mgmt`, `spint`) is in scope in the program policy. Many are out of scope.
2. **Origin check**: Akamai-fronted hosts may have a direct origin (e.g. `api.bridges`, Amazon Linux Apache). Check historical DNS and certificate transparency, only if allowed.
3. **Hybris (`commerce-api.direct*`)**: check `/hac`, `/occ/v2/<base>/`, `/rest/v2/`, anonymous product/cart endpoints.
4. **Next.js (`session-share`, `profile`)**: check `/_next/static/`, `.map` files, `__NEXT_DATA__`, middleware bypass by version.
5. **OAuth flows (`signin`, `help`, `learn`)**: check `redirect_uri`, `state`, PKCE, and open redirect.
6. **Java (`entitlements`)**: check `/actuator/*`, `/swagger`, `/v2/api-docs`.
7. **Header tests**: `Accept: application/json`, `X-Forwarded-For`, `Host` header on 406/403 responses.
8. **Re-run httpx with more fields** for better data: `httpx -l domains -sc -title -td -server -location -cname -ip -cdn -follow-redirects -silent -o httpx_full.txt`

## 4. Caveats

- The httpx file lacks a clean status-code column for most rows, so some statuses above are inferred from titles. Re-run with `-sc` for accurate codes.
- "Vulnerable with" entries are **test hypotheses**, not confirmed findings.
- Akamai blocks may cause false negatives. A "403" does not mean the endpoint is absent.
