# PlayStation Bug Bounty: Attack Plan

Split of the program into 10 test pieces, ordered by priority. Built from `domains`, `subs`, `.api_subs`, `commerce-api`, and `recon_table.md`.

**Before testing any piece:** confirm each host is in scope on the live HackerOne page (hackerone.com/playstation). The launch scope covered `*.playstation.net`, `*.sonyentertainmentnetwork.com`, `*.api.playstation.com`, `my`, `store`, `social`, `transact`, and `wallets.api`. Hosts outside that list may now be out of scope.

## Status

| # | Piece | Status | Findings | Notes |
|---|---|---|---|---|
| 1 | SSO / OAuth / login | Not started | 0 | |
| 2 | Money flow (prod) | Not started | 0 | |
| 3 | Account & identity APIs (prod) | Not started | 0 | |
| 4 | Entitlements & DRM | Not started | 0 | |
| 5 | Developer & partner portals | Not started | 0 | |
| 6 | Hybris / SAP Commerce | Not started | 0 | |
| 7 | Front-end SPAs (Next.js/React) | Not started | 0 | |
| 8 | Social, activity & community | Not started | 0 | |
| 9 | PSN / NP & activation | Not started | 0 | |
| 10 | Internal-looking & legacy | Not started | 0 | |
| P | Parked: analytics, ads, data | Not started | 0 | Endpoint discovery only |
| M | Parked: management tier | Not started | 0 | Likely out of scope |

Status values: Not started / In progress / Done / Out of scope.

---

## 1. SSO / OAuth / login (highest priority)

- [ ] Scope check: `signin.playstation.net`, `id.sonyentertainmentnetwork.com`, `account.sonyentertainmentnetwork.com`, `ca.account.sony.com`, `my.account.sony.com`, `help.playstation.net`, `learn.playstation.net`
- [ ] OAuth `redirect_uri` manipulation (allowlist bypass, path traversal, subdomain tricks)
- [ ] `state` parameter and PKCE validation
- [ ] Open redirect via `returnUrl`, `redirect`, `redirect_uri`, `next`
- [ ] Account enumeration on login and password reset
- [ ] Login CSRF
- [ ] `help.playstation.net/navpage.do` parameter handling, reflected XSS

## 2. Money flow (prod)

- [ ] Scope check: `store`, `checkout`, `cart.api`, `transact`, `wallets.api`, `api.direct`, `codes`
- [ ] Price and quantity tampering (including negative values)
- [ ] Coupon reuse and stacking
- [ ] IDOR on order, cart, and wallet IDs
- [ ] Race conditions on code redemption and wallet balance
- [ ] Payment callback validation
- [ ] Rate-limit bypass on `wallets.api` (429 seen during recon)

## 3. Account & identity APIs (prod)

- [ ] Scope check: `accounts.api`, `authentication`, `basicprofile`, `consent-admin`, `capone.idmapper`
- [ ] IDOR on account and profile IDs
- [ ] Mass assignment on profile and account update
- [ ] Auth on sub-resources (email, security settings)
- [ ] Verbose errors on 406 responses (retry with `Accept: application/json`)

## 4. Entitlements & DRM

- [ ] Scope check: `entitlements.api`, `core-entitlements`, `drm`, `dss`, `bps-api.software.eu`
- [ ] Entitlement bypass and IDOR
- [ ] Java backend: `/actuator/*`, `/swagger`, `/v2/api-docs` (`entitlements.api` is Java)
- [ ] Auth on license and token endpoints

## 5. Developer & partner portals

- [ ] Scope check: `partners`, `develop`, `dashboard`, `register`, `publish`, `unity`, `connect-dashboard`, `docs`, `analytics`
- [ ] IDOR on `dashboard.playstation.net/sb/<id>`
- [ ] Upload handling on `publish` (type checks, storage exposure)
- [ ] Reflected XSS and open redirect via `docs` `origin=` parameter
- [ ] Mass assignment on `register`
- [ ] Exposed storage paths on `analytics` and `segments`

## 6. Hybris / SAP Commerce

- [ ] Scope check: 16 `commerce-api.direct*` hosts (prod first, then non-prod)
- [ ] `/hac` admin console exposure and default credentials
- [ ] Anonymous access on `/occ/v2/<base>/` and `/rest/v2/`
- [ ] OAuth client misconfiguration on OCC

## 7. Front-end SPAs (Next.js/React)

- [ ] Scope check: `profile`, `session-share`, `social`, `compete`, `www.playstation.com`, `m.sonyentertainmentnetwork.com`, `music.sonyentertainmentnetwork.com`
- [ ] `/_next/static/` source maps (`.map` files)
- [ ] `__NEXT_DATA__` secrets and env vars
- [ ] Next.js middleware auth bypass (check version)
- [ ] XSS through profile fields

## 8. Social, activity & community

- [ ] Scope check: `activity.api`, `communities.api`, `critic`, `customermessagecenter`, `closure`, `gr-content`, `gr-reporting`, `gr-processor`
- [ ] IDOR on community, message, and review IDs
- [ ] Content injection and stored XSS
- [ ] Cache poisoning on `announcements`
- [ ] Rate limits on review and message endpoints

## 9. PSN / NP & activation

- [ ] Scope check: `ps5.np`, `web.np`, `activate`, `subs`, `announcements`, `manuals`, `codes`
- [ ] Activation code brute force and enumeration
- [ ] NP ticket and token handling
- [ ] Path traversal on `manuals`
- [ ] IDOR on subscription records

## 10. Internal-looking & legacy

- [ ] Scope check: `api.bridges`, `dev.api.bridges`, `dev.api.wrapup`, `api.ivaldi`, `helpsystem-proxy`, `cloudtest-tree2.nonprod`, `adaptorapi`
- [ ] Origin exposure: `/server-status`, `/server-info`, default paths (`api.bridges` runs Apache on Amazon Linux)
- [ ] `Host` header tricks on 403 and 406 responses
- [ ] Path handling on `dev.api.wrapup` ("Invalid URL" response)

## P. Parked: analytics, ads, data

- [ ] Endpoint discovery only: `ad-conf-svc`, `ads-poc`, `audience-segments-module`, `cba`, `cis`, `cosmos`, `dms`, `dora`, `eliot-ness`, `ets`, `facts`, `genome`, `gms`, `ns1-data`, `feature`, `garbo`, `bumblebee`
- [ ] Note: most return Access Denied behind Akamai, so treat results as endpoint names only

## M. Parked: management tier

- [ ] Confirm scope first. Likely out of scope.
- [ ] Hosts: every `*.mgmt.*`, `e1-pmgt`, `q1-pmgt`, `ems-admin.e1-pmgt`

---

## Rules for all pieces

- Confirm scope before each request.
- Akamai blocks can hide real results, so a 403 does not mean the endpoint is absent.
- Log every finding with: host, endpoint, reproduction steps, impact, and evidence.
- Check for duplicates in the HackerOne program before reporting.
