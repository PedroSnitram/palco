# Ticket Sites Analysis: Cache and Real-Time

How two real ticketing sites, Sympla and Eventim, use HTTP caching and real-time updates, compared with the decisions already made for Palco. This document records evidence and recommendations; it does not change any decision. Changes, if accepted, go to [`api-contract.md`](api-contract.md) separately.

## Contents

- [Method](#method)
- [Sympla](#sympla)
- [Eventim](#eventim)
- [Real-time](#real-time)
- [Conclusion](#conclusion)

## Method

- **Date:** 2026-10-30.
- **Browser:** Google Chrome, incognito window, not logged in to either site.
- **Recording:** each page was recorded in the DevTools Network tab and exported as a sanitized HAR file, which excludes cookies and `Authorization` headers. The HAR files are not in this repository.
- **Pages per site:** home, event page, search for `rock`, ticket selection (stopping before login), and home again in the same tab to see the browser cache at work.
- **Re-recording:** the event and ticket-selection pages were recorded again the same day, with a wait of about 2.5 minutes, and the new recordings replaced the first ones.
- **Events used:**
  - Sympla: event `122112` on Bileto, Sympla's platform for large events, for both the event page and ticket selection. A first recording of the main Sympla checkout, for the event "O Último Rock do Ano", is also kept, because it is the only Sympla recording that shows Queue-it.
  - Eventim: Simple Plan, for both the artist page and the event page.

### What the tables show

Each page has one table with the HTML document, the site's own data calls, and one sample each of JavaScript, CSS and image. The **Size column** is what DevTools displays: `(memory cache)` or `(disk cache)` means the browser served the file itself, with no network request; otherwise it shows the bytes transferred.

Left out on purpose:

- third-party requests: analytics, ads, chat, cookie consent and anti-fraud scripts;
- first-party requests used for tracking, bot detection, feature flags or session checks.

### Limitations

- A HAR file records when requests happen, not how long the page stayed open. The event and ticket-selection pages were kept open for about 2.5 minutes; the home and search pages for less, so polling with a long interval on those pages may have been missed.
- Sympla's first visit was not cold: many files already came from the disk cache, so the "home" and "home again" tables look alike.
- The Eventim ticket step recorded only the event page itself: sectors and prices are rendered in its HTML, and no other request happened before login.

## Sympla

### Summary

- **Infrastructure:** two CDN layers, Cloudflare in front (`cf-cache-status`) and Amazon CloudFront behind it (`x-cache`), plus an API gateway (Kong, visible in the `Via` header). The main APIs live under `www.sympla.com.br/api/...`, the same origin as the page, so they need no CORS preflight. Bileto is the exception: its APIs live on `bff-sales-api-cdn.bileto.sympla.com.br`.
- **HTML:** sent with `private, no-cache, no-store`, yet the first home page load was a Cloudflare `HIT` with `Age: 1227`. The CDN is configured to keep a copy at the edge regardless of the header the browser receives. The second visit was `DYNAMIC`.
- **Showcase data:** `public, max-age=30` with `Last-Modified`, served by Cloudflare (`HIT`), and answered with `304` on revalidation.
- **Repeated call:** `search/category-type` is requested 9 times on every home page load, with no caching.
- **Static files (Next.js):** hashed file names with `public, max-age=31536000` (one year), without `immutable`. After the first page, they always come from the browser cache.
- **Bileto (event page):**
  - the HTML comes from Amazon S3 through CloudFront, with no `Cache-Control`;
  - the bundles have hashed names but only `max-age=1800` (30 minutes); in the first recording, made after that time had expired, one page load caused 51 revalidations (`304`);
  - the event data comes from `bff-sales-api-cdn`, another origin, without `Cache-Control`; CloudFront served it from its cache in the first recording (`Age: 23`) but not in the second (`Miss`);
  - choosing tickets loads the seat map (`GET maps/seats`) and creates the cart immediately (`POST /api/v1/cart`); both calls go to another origin, so each one is preceded by a CORS preflight.
- **Search:** the results come inside the HTML (server-side rendering), with no separate data call.
- **Main Sympla checkout (first recording):**
  - the HTML carries `x-queueit-connector: cloudflare`, which means the **Queue-it virtual waiting room** is connected at the CDN;
  - data is loaded through GraphQL instead of REST;
  - the service fee comes from its own endpoint (`processorfee`, `max-age=60`).

### Home

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.sympla.com.br/` | 200 | `private, no-cache, no-store, max-age=0, must-revalidate` | Last-Modified | 1227 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 75.9 kB |
| Data | `www.sympla.com.br/api/discovery-bff/home?…` | 304 | `public, max-age=30` | Last-Modified | 50 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 2.5 kB |
| Data | `www.sympla.com.br/api/discovery-bff/home/highlights?…` | 200 | `public, max-age=30` | Last-Modified | 6 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 27.9 kB |
| Data | `www.sympla.com.br/api/discovery-bff/search/category-type` (×9) | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 6.8 kB |
| JS | `discovery-next.svc.sympla.com.br/_next/static/chunks/webpack-0cb69cb0eccfc2b4.js` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 90230 | cf-cache-status: HIT | (disk cache) |
| CSS | `discovery-next.svc.sympla.com.br/_next/static/css/7b82c94f28c933db.css` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 188938 | cf-cache-status: HIT | (disk cache) |
| Img | `images.sympla.com.br/67f04b82c8dff.png` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 1690 | cf-cache-status: HIT<br>x-cache: Hit from cloudfront | (memory cache) |

### Event page (Bileto)

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `bileto.sympla.com.br/event/122112?…` | 200 | — | ETag, Last-Modified | 12475 | x-cache: Hit from cloudfront | 3.9 kB |
| Data | `bff-sales-api-cdn.bileto.sympla.com.br/api/v1/events/122112` | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 4.7 kB |
| Data | `bff-sales-api-cdn.bileto.sympla.com.br/api/v1/maps/schematic/142386` | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 2.3 kB |
| JS | `web-assets.bileto.sympla.com.br/bundle.app.e8007fcb.js` | 200 | `public, max-age=1800` | ETag, Last-Modified | 284407 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 32.5 kB |
| Img | `web-assets.bileto.sympla.com.br/src/images/event/welcome.svg` | 200 | `public, max-age=1800` | ETag, Last-Modified | 1853656 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 4.4 kB |

### Search

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.sympla.com.br/eventos?…` | 200 | `private, no-cache, no-store, max-age=0, must-revalidate` | Last-Modified | — | cf-cache-status: MISS<br>x-cache: Miss from cloudfront | 77.0 kB |
| JS | `discovery-next.svc.sympla.com.br/_next/static/chunks/webpack-0cb69cb0eccfc2b4.js` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 90230 | cf-cache-status: HIT | (memory cache) |
| CSS | `discovery-next.svc.sympla.com.br/_next/static/css/7b82c94f28c933db.css` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 188938 | cf-cache-status: HIT | (memory cache) |
| Img | `images.sympla.com.br/67f04b82c8dff.png` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 1690 | cf-cache-status: HIT<br>x-cache: Hit from cloudfront | (memory cache) |

### Ticket selection (Bileto)

Bileto is a single-page app: choosing tickets loads no new HTML document, so this table has no Doc row. The `cart` call is a `POST`.

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Data | `bff-sales-api-cdn.bileto.sympla.com.br/api/v1/maps/seats/407420` | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 3.5 kB |
| Data | `bff-sales-api-cdn.bileto.sympla.com.br/api/v1/cart` | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 1.1 kB |
| JS | `web-assets.bileto.sympla.com.br/bundle.30.e8007fcb.js` | 200 | `public, max-age=1800` | ETag, Last-Modified | 284602 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 8.9 kB |
| Img | `web-assets.bileto.sympla.com.br/images/ico_wheelchair.png` | 200 | `public, max-age=1800` | ETag, Last-Modified | 2617809 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 849 B |

### Checkout (main Sympla site, first recording)

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.sympla.com.br/evento/o-ultimo-rock-do-ano/3591736/checkout/etapa-1?…` | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront<br>**x-queueit-connector: cloudflare** | 87.1 kB |
| Data | `apollo-server.svc.sympla.com.br/v1/graphql` (×3) | 200 | — | ETag | — | cf-cache-status: DYNAMIC | 1.6 kB |
| Data | `simple-gql-proxy.svc.sympla.com.br/v1/proxy/execute` | 200 | — | ETag | — | cf-cache-status: DYNAMIC | 1.3 kB |
| Data | `payments.svc.sympla.com.br/api/v1/processorfee/event/3591736/ref/{ref}?…` | 200 | `public, max-age=60` | — | — | cf-cache-status: DYNAMIC | 562 B |
| Data | `www.sympla.com.br/api/v1/checkout/event-risk?…` | 200 | `public, max-age=86400` | ETag | — | cf-cache-status: REVALIDATED<br>x-cache: Miss from cloudfront | 492 B |
| JS | `na-bundle.sympla.com.br/build-purchase/prod/v0.1.112/vendor.bundle.js` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 1242 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 1201.8 kB |
| Img | `images.sympla.com.br/6ad76fd83a3d7-xs.jpg` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 1799 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | (disk cache) |

### Home again

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.sympla.com.br/` | 200 | `private, no-cache, no-store, max-age=0, must-revalidate` | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 78.0 kB |
| Data | `www.sympla.com.br/api/discovery-bff/home?…` | 200 | `public, max-age=30` | Last-Modified | 47 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 3.0 kB |
| Data | `www.sympla.com.br/api/discovery-bff/home/highlights?…` | 200 | `public, max-age=30` | Last-Modified | 45 | cf-cache-status: HIT<br>x-cache: Miss from cloudfront | 28.0 kB |
| Data | `www.sympla.com.br/api/discovery-bff/search/category-type` (×9) | 200 | — | — | — | cf-cache-status: DYNAMIC<br>x-cache: Miss from cloudfront | 7.0 kB |
| JS | `discovery-next.svc.sympla.com.br/_next/static/chunks/webpack-0cb69cb0eccfc2b4.js` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 90230 | cf-cache-status: HIT | (memory cache) |
| CSS | `discovery-next.svc.sympla.com.br/_next/static/css/7b82c94f28c933db.css` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 188938 | cf-cache-status: HIT | (memory cache) |
| Img | `images.sympla.com.br/67f04b82c8dff.png` | 200 | `public, max-age=31536000` | ETag, Last-Modified | 1690 | cf-cache-status: HIT<br>x-cache: Hit from cloudfront | (disk cache) |

## Eventim

### Summary

- **Infrastructure:** the CDN is Akamai, identified by `Server-Timing: ak_p` and `X-Akamai-Transformed`. It exposes no cache status header, so edge hits cannot be seen from the browser.
- **HTML:** `no-cache` on every page. `no-cache` lets a copy be stored but requires checking with the server before every use; `no-store` forbids storing at all.
- **Queue-it:** `x-queueit-connector: akamai` appears on the artist page (in both recordings), on the event page (in the first recording only) and on the `eventseries` API, and never on the home or search pages. The header is not sent on every response, but it only shows up on the routes that lead to buying.
- **Search data:**
  - comes from a separate origin, `public-api.eventim.com`, with `max-age=30` and no `ETag` or `Last-Modified`;
  - each search makes 4 parallel calls, and each one is preceded by a CORS preflight (`OPTIONS`).
- **Static files:**
  - `max-age=600` to `900` (10 to 15 minutes), with `ETag` and `Last-Modified`;
  - most file names have no hash, for example `jquery.min.js`;
  - on the home page visit 15 minutes later, `patterns.css` transferred only 271 bytes: its 900 seconds had expired, and the browser had to check with the server again.
- **Tickets:** sectors and prices are rendered in the event page HTML, with no separate availability call. Each sector shows three prices: half price (meia-entrada), full price, and a discounted price for customers of a partner bank.
- **Bot detection:** first-party `POST` requests to an obfuscated path, a few seconds apart, together with `/akam/13/pixel_...`. This is consistent with Akamai Bot Manager collecting sensor data, not with product data.

### Home

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.eventim.com.br/` | 200 | `no-cache` | — | — | Akamai (no cache status header) | 67.2 kB |
| Data | `public-api.eventim.com/evi/api/evi/public/rule?…` (×2) | 200 | `no-cache, no-store, must-revalidate` | — | — | — | 2.7 kB |
| JS | `www.eventim.com.br/resources/{hash}` | 200 | `max-age=600` | ETag, Last-Modified | — | Akamai (no cache status header) | 74.9 kB |
| CSS | `www.eventim.com.br/obj/assets/toolkit/styles/patterns.css` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | 87.3 kB |
| Img | `www.eventim.com.br/obj/media/BR-eventim/teas…alegria-eventim-header.png` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | 187.6 kB |

### Artist page

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.eventim.com.br/artist/simple-plan/?…` | 200 | `no-cache` | — | — | Akamai (no cache status header)<br>**x-queueit-connector: akamai** | 58.1 kB |
| Data | `public-api.eventim.com/evi/api/evi/public/rule?…` | 200 | `no-cache, no-store, must-revalidate` | — | — | — | 2.8 kB |
| JS | `www.eventim.com.br/resources/{hash}` | 200 | `max-age=600` | ETag, Last-Modified | — | Akamai (no cache status header) | (disk cache) |
| CSS | `www.eventim.com.br/obj/assets/toolkit/styles/patterns.css` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (disk cache) |
| Img | `www.eventim.com.br/obj/media/BR-eventim/specialLogos/checkoutApp/logo.svg` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (disk cache) |

### Search

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.eventim.com.br/search/?…` | 200 | `no-cache` | — | — | Akamai (no cache status header) | 40.3 kB |
| Data | `public-api.eventim.com/websearch/search/api/exploration/v1/products?…` | 200 | `max-age=30` | — | — | — | 6.0 kB |
| Data | `public-api.eventim.com/websearch/search/api/exploration/v1/productGroups?…` | 200 | `max-age=30` | — | — | — | 5.4 kB |
| Data | `public-api.eventim.com/websearch/search/api/exploration/v1/attractions?…` | 200 | `max-age=30` | — | — | — | 3.1 kB |
| Data | `public-api.eventim.com/websearch/search/api/exploration/v1/locations?…` | 200 | `max-age=30` | — | — | — | 3.7 kB |
| Data | `www.eventim.com.br/api/eventseries/additionalInfo?…` | 200 | `no-cache` | — | — | Akamai (no cache status header)<br>**x-queueit-connector: akamai** | 2.4 kB |
| JS | `www.eventim.com.br/resources/{hash}` | 200 | `max-age=600` | ETag, Last-Modified | — | Akamai (no cache status header) | (memory cache) |
| CSS | `www.eventim.com.br/obj/assets/toolkit/styles/patterns.css` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (memory cache) |
| Img | `www.eventim.com.br/obj/media/BR-eventim/specialLogos/checkoutApp/logo.svg` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (memory cache) |

### Event page (ticket selection)

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.eventim.com.br/event/simple-plan-the-big…-tour-qualistage-22327205/?…` | 200 | `no-cache` | — | — | Akamai (no cache status header) | 65.8 kB |
| JS | `www.eventim.com.br/resources/{hash}` | 200 | `max-age=600` | ETag, Last-Modified | — | Akamai (no cache status header) | (memory cache) |
| CSS | `www.eventim.com.br/obj/assets/toolkit/styles/patterns.css` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (memory cache) |
| Img | `www.eventim.com.br/obj/media/BR-eventim/specialLogos/checkoutApp/logo.svg` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (memory cache) |

### Home again

| Type | Request | Status | Cache-Control | ETag / Last-Modified | Age | CDN | Size column |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doc | `www.eventim.com.br/` | 200 | `no-cache` | — | — | Akamai (no cache status header) | 66.2 kB |
| Data | `public-api.eventim.com/evi/api/evi/public/rule?…` (×2) | 200 | `no-cache, no-store, must-revalidate` | — | — | — | 2.7 kB |
| JS | `www.eventim.com.br/resources/{hash}` | 200 | `max-age=600` | ETag, Last-Modified | — | Akamai (no cache status header) | (disk cache) |
| CSS | `www.eventim.com.br/obj/assets/toolkit/styles/patterns.css` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | 271 B |
| Img | `www.eventim.com.br/obj/media/BR-eventim/teas…rd/2020/teatros-header.gif` | 200 | `max-age=900` | ETag, Last-Modified | — | Akamai (no cache status header) | (disk cache) |

## Real-time

| Site | WebSocket | SSE | Polling of product data | Repeated requests that are not product data |
| --- | --- | --- | --- | --- |
| Sympla | None | None | None | Analytics (Mixpanel, TikTok, Google Analytics), every 5 to 15 s |
| Eventim | None | None | None | Bot detection (3 `POST`s per page, 1 to 17 s apart) and server-side analytics (`sst.eventim.com.br`) |

On the event pages, other requests kept arriving until 159 s (Sympla) and 184 s (Eventim) after loading, but the last request for product data happened within the first 6 seconds. On the ticket-selection pages, no request at all happened after 18 seconds.

Neither site updates ticket availability while the page is open. Availability is read when the page loads: in the event page HTML on Eventim, through the seat map call on Bileto, and through GraphQL calls on the main Sympla checkout.

## Conclusion

| Palco decision | What was observed | Verdict |
| --- | --- | --- |
| Showcase data (list and detail) cached with `public, max-age=60` and `ETag` | Both sites cache showcase data for 30 s: Sympla at the CDN (`HIT`), Eventim in the browser. Sympla validates with `Last-Modified`; no data call used `ETag`. | **Confirms** a short public cache. Revisit 30 s versus 60 s with load tests in Module 21. |
| Static files: no decision yet | Sympla: hashed names cached for one year, with no network on later pages. Bileto and Eventim: 10 to 30 minutes, followed by repeated revalidations. | **Suggests a decision:** `public, max-age=31536000, immutable` for hashed files (the Angular build generates hashed names), and `no-cache` for `index.html`, so that new deploys are picked up. |
| Ticket availability by polling (Lesson 6) | No polling on either site during about 2.5 minutes on the event and ticket-selection pages. Availability is read when the page loads, and Bileto creates the cart as soon as tickets are chosen. | **Does not confirm.** Re-evaluate in the cart module: reading availability on load and checking it again when the tickets are reserved may be enough. |
| SSE in the waiting room (Module 22) | No SSE. Both sites use Queue-it, connected at the CDN (Cloudflare on Sympla, Akamai on Eventim), only on the routes that lead to buying. The queue page itself was not observed, because no queue was active. | **No evidence about SSE.** Suggests studying edge waiting rooms, and building versus buying one, in Module 22. |
| WebSocket for check-in | The staff check-in apps are not public. | **No evidence.** |
| API on the same origin as the site, through a gateway or proxy (Lesson 5) | Sympla's main site: APIs on the same origin, with no preflight. Bileto and Eventim's search: APIs on another origin, with a preflight before each call. | **Confirms.** |

### Suggestions for the API contract

These are suggestions only; any change is made in [`api-contract.md`](api-contract.md) separately.

- **Age rating:** Eventim shows the age rating (classificação etária) on the event page, and Brazilian law (ECA, art. 74) requires public shows to inform it. The contract has no field for it.
- **More price types:** Eventim sells a partner-bank price alongside full and half price. `TicketOption.type` only allows `full` and `half`, and the contract does not yet say whether new types can be added in `v1`.
