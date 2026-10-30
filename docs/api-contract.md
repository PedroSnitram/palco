# Palco API Contract: Showcase (v1)

This document describes how the public website (`palco-site`) requests showcase data from the API and what the API returns. Both sides follow it: the frontend, the mock API (json-server) and the real API (NestJS). If the code and this document disagree, this document wins until it is changed on purpose.

**Status:** draft, written before any application code.

**In scope:** listing events, searching events and viewing event details.

**Out of scope:** cart, checkout and payment; authentication; organizer endpoints (creating, editing or deleting events); check-in and real-time updates; how the server is implemented.

## Contents

- [Conventions](#conventions)
- [Shared objects](#shared-objects)
- [GET /v1/events](#get-v1events)
- [GET /v1/events/{slug}](#get-v1eventsslug)
- [Error format](#error-format)
- [Changing this contract](#changing-this-contract)

## Conventions

### Base URL and versioning

All paths start with `/v1`. The host is not defined yet; examples use relative paths.

### Format

- Request and response bodies are JSON, sent as `Content-Type: application/json; charset=utf-8`.
- Error bodies use `Content-Type: application/problem+json` (see [Error format](#error-format)).
- Field names are in English and camelCase: `startsAt`, `priceFrom`.
- Every field listed in this document is always present in the response. A field with no value is `null`; it is never omitted.
- Clients must ignore fields they do not know. This allows the API to add fields without breaking anyone.

### Identifiers

- `id` is a string and never changes. Clients treat it as opaque: no math, no assumptions about its format.
- `slug` is a readable identifier made of lowercase letters `a-z`, digits and hyphens, for example `festival-de-verao-2027`. It is unique, but it can change when an event is renamed (see [Old slugs](#old-slugs)).

### Dates and times

- Date-times use ISO 8601 with the UTC offset of the venue's local time: `2027-01-16T16:00:00-03:00`.
- Date-only query parameters use `YYYY-MM-DD` and refer to the venue's local calendar date.

### Money

Money is always an object. Amounts are integers in cents, never decimals.

```json
{ "amountInCents": 15000, "currency": "BRL" }
```

`currency` is an ISO 4217 code. Clients format the value for display (`R$ 150,00`) and add amounts in cents. An `amountInCents` of `0` means free.

### Pagination

Collections accept `page` and `pageSize` and return the items in `data` and the pagination details in `meta`:

```json
{
  "data": [],
  "meta": { "page": 2, "pageSize": 20, "totalItems": 134, "totalPages": 7 }
}
```

| Parameter | Type | Default | Rules |
| --- | --- | --- | --- |
| `page` | integer | `1` | 1 or greater |
| `pageSize` | integer | `20` | 1 to 100 |

A `page` beyond the last page returns `200 OK` with an empty `data` array.

### Categories

Categories are a fixed list. The API sends only the slug; the frontend translates it for display.

| Slug | Meaning |
| --- | --- |
| `concerts` | Concerts and shows |
| `festivals` | Music festivals |
| `theater` | Theater, dance and musicals |
| `comedy` | Stand-up and comedy |
| `kids` | Children's events |
| `sports` | Sports |

### Event status

| Status | Meaning |
| --- | --- |
| `scheduled` | The event is announced, but sales have not opened yet. `salesStartAt` says when they open. |
| `on_sale` | Tickets are on sale. |
| `few_left` | Tickets are on sale and almost gone. The threshold is decided by the server. |
| `sold_out` | No tickets left. |
| `cancelled` | The event will not happen. |
| `postponed` | The event will not happen on the original date and the new date is not announced yet. `startsAt` keeps the original date. |

## Shared objects

### EventSummary

Used in the event list. It contains only what an event card needs.

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Permanent identifier. |
| `slug` | string | Readable identifier used in URLs. |
| `name` | string | Event name. |
| `startsAt` | date-time | When the event starts. |
| `endsAt` | date-time | When the event ends. Lets the card show multi-day events as a date range. |
| `salesStartAt` | date-time | When ticket sales open (or opened). |
| `status` | string | One of the [event statuses](#event-status). |
| `category` | string | One of the [category slugs](#categories). |
| `image` | Image | Cover image. |
| `venue` | VenueSummary | Where the event happens. |
| `priceFrom` | Money or `null` | Lowest full-price ticket among the options that are not `sold_out` or `closed`, without the service fee. `null` when there is none. An amount of `0` means the event is free. |

`priceFrom` ignores half-price tickets on purpose: half price requires proof of eligibility, so showing it as the starting price would mislead most buyers.

### EventDetail

Used in the event detail. It contains every field of `EventSummary`, plus:

| Field | Type | Description |
| --- | --- | --- |
| `description` | string | Plain text written by the organizer. Line breaks are `\n`. It never contains HTML. |
| `venue` | Venue | Replaces `VenueSummary` with the full venue, including the address. |
| `ticketOptions` | TicketOption[] | Every ticket that can be bought for the event. |

### Image

| Field | Type | Description |
| --- | --- | --- |
| `url` | string | Absolute URL of the image. |
| `alt` | string | Text alternative read by screen readers, describing the image. |

### VenueSummary

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Venue name. |
| `city` | City | City of the venue. |

### Venue

Every field of `VenueSummary`, plus:

| Field | Type | Description |
| --- | --- | --- |
| `address` | Address | Street address. |

### City

| Field | Type | Description |
| --- | --- | --- |
| `slug` | string | Used by the `city` filter, for example `sao-paulo`. |
| `name` | string | Display name, for example `São Paulo`. |
| `state` | string | Two-letter Brazilian state code, for example `SP`. |

### Address

| Field | Type | Description |
| --- | --- | --- |
| `street` | string | Street name. |
| `number` | string | Building number. A string because values such as `s/n` exist. |
| `complement` | string or `null` | Extra details. |
| `neighborhood` | string | Neighborhood. |
| `postalCode` | string | Brazilian CEP in the format `00000-000`. |

### TicketOption

One ticket that can be bought: a sector, in a batch, of a type. For example, "Pista, 1º lote, meia-entrada".

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | Permanent identifier. The cart will use it in a later version of this contract. |
| `sector` | string | Sector name, for example `Pista` or `Camarote`. |
| `batch` | string | Batch name, for example `1º lote`. |
| `type` | string | `full` for full price (inteira) or `half` for half price (meia-entrada, required by Brazilian law 12.933/2013). |
| `price` | Money | Ticket price, without the service fee. |
| `serviceFee` | Money | Service fee charged on top of the price. It is always shown to the buyer before purchase. |
| `status` | string | One of the ticket option statuses below. |

| Ticket option status | Meaning |
| --- | --- |
| `upcoming` | Not on sale yet, for example a second batch that opens when the first one sells out. |
| `on_sale` | On sale. |
| `few_left` | On sale and almost gone. |
| `sold_out` | No tickets left. |
| `closed` | Sales for this option ended without selling out, for example because the batch's sales period ended. |

## GET /v1/events

Lists upcoming events. The same endpoint is used for search: a search is the list with filters.

### Parameters

All parameters are optional query parameters and can be combined.

| Parameter | Type | Description |
| --- | --- | --- |
| `q` | string | Text search, 2 to 100 characters. |
| `city` | string | City slug, for example `sao-paulo`. |
| `category` | string | One of the [category slugs](#categories). |
| `dateFrom` | date | Only events that start on or after this date. |
| `dateTo` | date | Only events that start on or before this date. |
| `page` | integer | See [Pagination](#pagination). |
| `pageSize` | integer | See [Pagination](#pagination). |

### Rules

- Only events that have not ended yet are listed. Cancelled and postponed events are still listed, with their status, so people can find out what happened.
- Events are sorted by `startsAt`, earliest first. Events that start at the same time are sorted by `id`, so pages never repeat or skip an event.
- Text search (`q`):
  - matches the event name, the venue name and the city name;
  - ignores accents and letter case: `sao paulo` finds `São Paulo`;
  - treats every character as plain text: characters such as `%`, `'`, `"` or `*` never cause an error;
  - an empty `q` is ignored.
- A `city` slug that does not exist returns an empty list, not an error.
- Unknown query parameters are ignored.

### Success response

`200 OK`

```http
Content-Type: application/json; charset=utf-8
Cache-Control: public, max-age=60
```

Example: `GET /v1/events?city=sao-paulo&page=1&pageSize=2`

```json
{
  "data": [
    {
      "id": "1842",
      "slug": "festival-de-verao-2027",
      "name": "Festival de Verão 2027",
      "startsAt": "2027-01-16T16:00:00-03:00",
      "endsAt": "2027-01-17T04:00:00-03:00",
      "salesStartAt": "2026-11-10T10:00:00-03:00",
      "status": "on_sale",
      "category": "festivals",
      "image": {
        "url": "https://images.palco.example/events/festival-de-verao-2027.jpg",
        "alt": "Crowd in front of a lit stage at sunset"
      },
      "venue": {
        "name": "Arena Norte",
        "city": { "slug": "sao-paulo", "name": "São Paulo", "state": "SP" }
      },
      "priceFrom": { "amountInCents": 15000, "currency": "BRL" }
    },
    {
      "id": "1907",
      "slug": "noite-de-stand-up",
      "name": "Noite de Stand-up",
      "startsAt": "2027-02-05T21:00:00-03:00",
      "endsAt": "2027-02-05T23:00:00-03:00",
      "salesStartAt": "2026-12-01T12:00:00-03:00",
      "status": "scheduled",
      "category": "comedy",
      "image": {
        "url": "https://images.palco.example/events/noite-de-stand-up.jpg",
        "alt": "Comedian holding a microphone under a spotlight"
      },
      "venue": {
        "name": "Teatro Jardim",
        "city": { "slug": "sao-paulo", "name": "São Paulo", "state": "SP" }
      },
      "priceFrom": { "amountInCents": 8000, "currency": "BRL" }
    }
  ],
  "meta": { "page": 1, "pageSize": 2, "totalItems": 37, "totalPages": 19 }
}
```

### Errors

| Status | When |
| --- | --- |
| `400 Bad Request` | A parameter is invalid: `q` shorter than 2 or longer than 100 characters, `page` or `pageSize` out of range, an unknown `category`, a malformed date, or `dateFrom` after `dateTo`. |
| `429 Too Many Requests` | The client sent too many requests. See [Error format](#error-format). |

## GET /v1/events/{slug}

Returns the full details of one event.

### Parameters

| Parameter | In | Type | Description |
| --- | --- | --- | --- |
| `slug` | path | string | The event slug, for example `festival-de-verao-2027`. |

### Success response

`200 OK`

```http
Content-Type: application/json; charset=utf-8
Cache-Control: public, max-age=60
ETag: "a1b2c3"
```

Example: `GET /v1/events/festival-de-verao-2027`

```json
{
  "id": "1842",
  "slug": "festival-de-verao-2027",
  "name": "Festival de Verão 2027",
  "description": "Two stages and twelve hours of music.\nGates open at 2 p.m.",
  "startsAt": "2027-01-16T16:00:00-03:00",
  "endsAt": "2027-01-17T04:00:00-03:00",
  "salesStartAt": "2026-11-10T10:00:00-03:00",
  "status": "on_sale",
  "category": "festivals",
  "image": {
    "url": "https://images.palco.example/events/festival-de-verao-2027.jpg",
    "alt": "Crowd in front of a lit stage at sunset"
  },
  "venue": {
    "name": "Arena Norte",
    "city": { "slug": "sao-paulo", "name": "São Paulo", "state": "SP" },
    "address": {
      "street": "Avenida das Nações",
      "number": "1000",
      "complement": null,
      "neighborhood": "Barra Funda",
      "postalCode": "01156-000"
    }
  },
  "priceFrom": { "amountInCents": 15000, "currency": "BRL" },
  "ticketOptions": [
    {
      "id": "1842-01",
      "sector": "Pista",
      "batch": "1º lote",
      "type": "full",
      "price": { "amountInCents": 15000, "currency": "BRL" },
      "serviceFee": { "amountInCents": 2250, "currency": "BRL" },
      "status": "few_left"
    },
    {
      "id": "1842-02",
      "sector": "Pista",
      "batch": "1º lote",
      "type": "half",
      "price": { "amountInCents": 7500, "currency": "BRL" },
      "serviceFee": { "amountInCents": 1125, "currency": "BRL" },
      "status": "on_sale"
    },
    {
      "id": "1842-03",
      "sector": "Pista",
      "batch": "2º lote",
      "type": "full",
      "price": { "amountInCents": 18000, "currency": "BRL" },
      "serviceFee": { "amountInCents": 2700, "currency": "BRL" },
      "status": "upcoming"
    },
    {
      "id": "1842-04",
      "sector": "Camarote",
      "batch": "1º lote",
      "type": "full",
      "price": { "amountInCents": 42000, "currency": "BRL" },
      "serviceFee": { "amountInCents": 6300, "currency": "BRL" },
      "status": "sold_out"
    }
  ]
}
```

### Conditional requests

The response has an `ETag`. When the client sends it back in `If-None-Match` and the event has not changed, the API answers `304 Not Modified` with no body, and the client reuses its copy.

```http
GET /v1/events/festival-de-verao-2027
If-None-Match: "a1b2c3"
```

```http
HTTP/1.1 304 Not Modified
ETag: "a1b2c3"
```

### Old slugs

When an event is renamed, its slug can change. Requests to an old slug are redirected permanently to the current one, so shared links keep working:

```http
HTTP/1.1 301 Moved Permanently
Location: /v1/events/festival-de-verao-2027-edicao-extra
```

### Errors

| Status | When |
| --- | --- |
| `404 Not Found` | No event has this slug, now or in the past. |
| `429 Too Many Requests` | The client sent too many requests. See [Error format](#error-format). |

## Error format

Every error response uses Problem Details ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) with `Content-Type: application/problem+json`.

| Field | Type | Description |
| --- | --- | --- |
| `type` | string | URI that identifies the kind of problem. |
| `title` | string | Short, fixed summary of the kind of problem. |
| `status` | integer | Same value as the HTTP status code. |
| `detail` | string | Explanation of this specific occurrence. |
| `errors` | array | Only on `400`: one item per invalid parameter, with `parameter` and `message`. |

`palco.example` in the `type` URIs is a placeholder until the production domain exists.

### Examples

`404 Not Found`

```json
{
  "type": "https://palco.example/problems/not-found",
  "title": "Resource not found",
  "status": 404,
  "detail": "No event with slug 'show-que-nao-existe'."
}
```

`400 Bad Request`

```json
{
  "type": "https://palco.example/problems/invalid-parameters",
  "title": "Invalid parameters",
  "status": 400,
  "detail": "2 query parameters are invalid.",
  "errors": [
    { "parameter": "pageSize", "message": "Must be between 1 and 100." },
    { "parameter": "category", "message": "Must be one of: concerts, festivals, theater, comedy, kids, sports." }
  ]
}
```

`429 Too Many Requests`

The response includes a `Retry-After` header with the number of seconds the client must wait before trying again.

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 30
```

```json
{
  "type": "https://palco.example/problems/too-many-requests",
  "title": "Too many requests",
  "status": 429,
  "detail": "Try again in 30 seconds."
}
```

## Changing this contract

- **Allowed in `v1`:** adding new fields, new optional parameters, new categories or new statuses. Clients ignore what they do not know.
- **Requires `v2`:** removing or renaming a field, changing a field's type or meaning, or making an optional parameter required.
- Every change is made in this document first, in the same commit as the code that implements it.
