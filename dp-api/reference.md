# DP API reference

One API for distribution partners such as mobility apps, fleets and car makers: find chargers,
show current availability and prices, start and stop charging, follow every session live.

> Test first in the [Sandbox](/sandbox.html). Simulated stations, no real money.

| | |
| --- | --- |
| Base URL | `https://dp.networkcore.org` |
| Auth | `Authorization: Bearer <api_key>` |
| Format | JSON, amounts incl. VAT, ISO 8601 UTC |

## Integration levels

- **Data**: locations, current availability and prices. No money moves and no sessions start. Fits
  maps, POI display and route planners.
- **Full charging**: everything in Data, plus starting and stopping sessions and live session
  events.

A read API key gives Data, a write key gives Full charging. Keys start with `nc_live_`, or
`nc_test_` for the [Sandbox](/sandbox.html).

| Group | Data | Full charging |
| --- | --- | --- |
| [Locations and availability](#locations-and-availability) | ✓ | ✓ |
| [Prices](#prices) | ✓ | ✓ |
| [Data webhooks](#data-webhooks) | ✓ | ✓ |
| [Sessions](#sessions) | | ✓ |
| [Charging webhooks and live stream](#charging-webhooks-and-live-stream) | | ✓ |
| [Receipts and tax invoices](#receipts-and-tax-invoices) | | ✓ |
| [Disputes](#disputes) | | ✓ |
| [Payments (optional)](#payments) | | ✓ |

## Conventions

### Your reference

Every session carries `reference`: your own order or transaction ID. It is required, unique per
partner, and comes back on every session, webhook, tax invoice and dispute. Sending the same
`reference` twice replays the first answer instead of starting a second charge.

### Charging partner

The company that runs the chargers. Locations show it as `charging_partner`.

### Charge point status

Every charge point has one of these statuses, passed on as the charging partner reports it. It can
lag by up to about 30 seconds.

| Status | Meaning |
| --- | --- |
| `AVAILABLE` | Free to start a charge |
| `CHARGING` | A charge is running |
| `BLOCKED` | Occupied without charging, for example a parked vehicle |
| `RESERVED` | Reserved for a driver |
| `INOPERATIVE` | Not working at the moment |
| `OUTOFORDER` | Out of order |
| `PLANNED` | Planned for the future |
| `UNKNOWN` | Status not known |
| `REMOVED` | Removed by the charging partner. Never returned: the charge point no longer appears. |

### Errors

One shape everywhere:

```json
{ "error": { "code": "CHARGE_POINT_OCCUPIED",
             "message": "Charge point EVSE-01 already has an active session." } }
```

A missing or invalid key returns `401 UNAUTHORIZED`. A read key on a write endpoint returns
`403 FORBIDDEN`.

### Headers

- `X-Correlation-ID` on every response. Quote it when you report a problem.
- `Idempotent-Replayed: true` when an answer is a replay of an earlier request.

### Lists and syncing

- `limit` and `offset`, with `total_count` in the response.
- `updated_since` returns only what changed, for cheap delta syncs.

### Webhooks

**Setup.** Ask NetworkCore to register one HTTPS endpoint for you. You get a signing secret (starts
with `whsec_`), shown once. All events go to that endpoint.

**Payload.** Every event has `id`, `type`, `created_at` and `data`. Session events carry the same
body as `GET /sessions/{id}`:

```json
{
  "id": "evt_01J9XK2",
  "type": "session.completed",
  "created_at": "2026-10-01T16:12:44Z",
  "data": {
    "id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "status": "COMPLETED",
    "location_id": "LOC-CDMX-0142",
    "charge_point_id": "MX*EXA*E0142A",
    "connector_id": "1",
    "start_time": "2026-10-01T15:30:41Z", "end_time": "2026-10-01T16:12:30Z",
    "kwh": 40.2, "currency": "MXN",
    "total_cost": 372.78, "total_cost_status": "FINAL",
    "receipt_url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/receipt",
    "tax_invoice": { "status": "ISSUED", "url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/tax_invoice" },
    "created_at": "2026-10-01T15:30:02Z",
    "updated_at": "2026-10-01T16:12:44Z"
  }
}
```

**Verifying signatures.** We follow the Standard Webhooks spec.
Each request carries three headers: `webhook-id`, `webhook-timestamp` and `webhook-signature`. The
signature is an HMAC-SHA256 of `{webhook-id}.{webhook-timestamp}.{raw body}` using your secret,
base64-encoded and prefixed with `v1,`. Use the official Standard Webhooks library for your
language, or compute it yourself. Reject requests whose timestamp is more than 5 minutes old.

```js
import { Webhook } from "standardwebhooks";
const wh = new Webhook(process.env.NETWORKCORE_WEBHOOK_SECRET);
const event = wh.verify(rawBody, headers); // throws if the signature is invalid
```

**Delivery rules.**

- Respond with any `2xx` within 10 seconds. Do slow work after responding.
- We retry failed deliveries with increasing delays for up to 24 hours.
- Delivery is at least once: use `webhook-id` to ignore duplicates.
- Order is not guaranteed: compare `updated_at`, or read the session with `GET /sessions/{id}`.
- Secret rotation: when the secret is rotated, old and new secrets both work for 24 hours.

### Settlement and reconciliation

Every session carries your `reference` and its final amounts, so you can match it in your own
system. Settlements and session detail are in the NetworkCore portal, with CSV export. A Settlement
API is available on request.

### Examples

IDs, tax IDs and amounts in examples are illustrative.

### Session lifecycle

```
PENDING → ACTIVE → COMPLETED
        ↘ FAILED
```

| Status | Meaning |
| --- | --- |
| `PENDING` | We sent the start to the charger and are waiting for it to confirm |
| `ACTIVE` | Energy is flowing |
| `COMPLETED` | The charger's final record (CDR) arrived and the final amounts are known |
| `FAILED` | The charger refused or never confirmed |

---

## Locations and availability

### `GET /locations`

Chargers near a point or inside a map area, with current status.

<details markdown="1">
<summary>Optional parameters</summary>

| Query | Meaning |
| --- | --- |
| `lat`, `lng`, `radius_km` | Search around a point |
| `bbox` | Or a map area: `min_lng,min_lat,max_lng,max_lat` |
| `country` | ISO code, e.g. `MX` |
| `min_power_kw` | e.g. `50` for fast charging |
| `connector` | `CCS2`, `TYPE2`, `CHADEMO`, `GBT`… |
| `available_only` | `true` to match only `AVAILABLE` charge points |
| `updated_since` | Only locations changed since this time |
| `limit`, `offset` | Paging |

</details>

```json
{
  "data": [{
    "id": "LOC-CDMX-0142",
    "name": "Example Plaza, Level 2",
    "charging_partner": { "id": "MX*EXA", "name": "Example CPO" },
    "address": "Av. Ejemplo 100, Ciudad de México",
    "coordinates": { "lat": 19.4402, "lng": -99.2038 },
    "country": "MX", "time_zone": "America/Mexico_City",
    "charge_points": [{
      "id": "MX*EXA*E0142A",
      "status": "AVAILABLE",
      "payment_url": "https://pay.example.com/c/LOC-CDMX-0142/MX*EXA*E0142A",
      "connectors": [{ "id": "1", "standard": "CCS2", "power_kw": 120, "tariff_id": "EXA-DC-PUB" }]
    }],
    "last_updated": "2026-10-01T15:04:11Z"
  }],
  "total_count": 1, "limit": 50, "offset": 0
}
```

`payment_url` is a link where a driver can pay for and start a charging session at this charge
point, or `null` when payment is not available for it. Use the URL as published; do not build or
rewrite it.

### `GET /locations/{id}`

One location with all its charge points, connectors, status and tariffs. Same shape as an item in
`GET /locations`.

Errors: `LOCATION_NOT_FOUND`

### `GET /availability`

Light status feed for keeping a map fresh without re-downloading locations. `status` uses the
same values as [Charge point status](#charge-point-status).

| Query | Meaning |
| --- | --- |
| `updated_since` **required** | Status changes since this time |

<details markdown="1">
<summary>Optional parameters</summary>

| Query | Meaning |
| --- | --- |
| `limit`, `offset` | Paging |

</details>

```json
{
  "data": [
    { "location_id": "LOC-CDMX-0142", "charge_point_id": "MX*EXA*E0142A",
      "status": "CHARGING", "changed_at": "2026-10-01T15:21:40Z" }
  ],
  "total_count": 1, "limit": 500, "offset": 0
}
```

## Prices

### `GET /tariffs`

Every CPO tariff on the network, as structured price components. Prices are what you pay under
your agreement, VAT included.

<details markdown="1">
<summary>Optional parameters</summary>

| Query | Meaning |
| --- | --- |
| `updated_since` | Only tariffs changed since this time |
| `limit`, `offset` | Paging |

</details>

```json
{
  "data": [{
    "id": "EXA-DC-PUB",
    "currency": "MXN",
    "elements": [{
      "components": [
        { "type": "ENERGY", "price": 8.90, "unit": "kWh" },
        { "type": "FLAT",   "price": 15.00, "unit": "session" }
      ],
      "restrictions": { "start_time": "06:00", "end_time": "22:00", "day_of_week": null }
    }],
    "last_updated": "2026-09-28T10:00:00Z"
  }],
  "total_count": 38, "limit": 50, "offset": 0
}
```

### `GET /tariffs/{id}`

One tariff. Same shape as an item in `GET /tariffs`.

Errors: `TARIFF_NOT_FOUND`

### `POST /quotes`

What a charge will cost your driver at this connector, with your agreement applied.

| Body field | Meaning |
| --- | --- |
| `location_id` **required** | |
| `charge_point_id` **required** | The charge point's `id` as returned by `GET /locations` |
| `connector_id` **required** | |
| `estimate.kwh` | Expected energy, e.g. `40`. Needed when the price is per kWh. |
| `estimate.minutes` | Expected duration. Needed for time-based prices. |

```json
{
  "data": {
    "currency": "MXN",
    "pricing_mode": "PUBLIC",
    "total_incl_vat": 371.00,
    "breakdown": [
      { "type": "ENERGY", "quantity": 40, "unit": "kWh", "amount": 356.00 },
      { "type": "FLAT",   "quantity": 1,  "unit": "session", "amount": 15.00 }
    ],
    "valid_until": "2026-10-01T16:00:00Z"
  }
}
```

`pricing_mode` is `PUBLIC`, `PUBLIC_DISCOUNT`, `PUBLIC_FIX` or `WHOLESALE`.

## Data webhooks

| Event | When |
| --- | --- |
| `evse.status_changed` | A charge point's status changed. Same item as `GET /availability`. |
| `location.updated` | A location was added, changed or removed. Carries `location_id`; read `GET /locations/{id}`. |
| `tariff.updated` | A CPO changed a price. Carries `tariff_id`; read `GET /tariffs/{id}`. |

---

## Sessions

### `POST /sessions`

Start charging. We send the start command to the charger.

| Body field | Meaning |
| --- | --- |
| `reference` **required** | Your order ID. Unique per partner. Same reference = same answer. |
| `location_id` **required** | |
| `charge_point_id` **required** | The charge point's `id` as returned by `GET /locations` |
| `connector_id` **required** | |
| `driver_ref` **required** | Your stable ID for the driver |

<details markdown="1">
<summary>Optional fields</summary>

| Body field | Meaning |
| --- | --- |
| `billing_details` | Send the driver's tax data and the tax invoice is issued automatically when the session completes. Same fields as [`PUT /sessions/{id}/billing_details`](#put-sessionsidbilling_details). |

</details>

```json
HTTP 201
{
  "data": {
    "id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "status": "PENDING",
    "location_id": "LOC-CDMX-0142",
    "charge_point_id": "MX*EXA*E0142A",
    "connector_id": "1",
    "start_time": null, "end_time": null,
    "kwh": 0, "currency": "MXN", "total_cost": null, "total_cost_status": null,
    "created_at": "2026-10-01T15:30:02Z",
    "updated_at": "2026-10-01T15:30:02Z"
  }
}
```

| Error | HTTP |
| --- | --- |
| `MISSING_FIELD` | 422 |
| `DUPLICATE_REFERENCE` (same `reference`, different request) | 422 |
| `NO_AGREEMENT` | 422 |
| `NO_TARIFF` | 422 |
| `LOCATION_NOT_FOUND` | 404 |
| `CHARGE_POINT_NOT_FOUND` | 404 |
| `CONNECTOR_NOT_FOUND` | 404 |
| `CHARGE_POINT_OCCUPIED` | 409 |
| `LOCATION_BLOCKED` | 409 |
| `REQUEST_IN_PROGRESS` | 409 |
| `TOKEN_ERROR` (driver could not be authorized at the charging partner) | 502 |
| `SESSION_START_FAILED` | 502 |

### `GET /sessions/{id}`

One session, live: energy so far, cost so far, status. `total_cost_status` is `ESTIMATED` before the
charger's final record, `FINAL` once it is confirmed, or `IN_REVIEW` while it is checked.

```json
{
  "data": {
    "id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "status": "COMPLETED",
    "location_id": "LOC-CDMX-0142",
    "charge_point_id": "MX*EXA*E0142A",
    "connector_id": "1",
    "start_time": "2026-10-01T15:30:41Z", "end_time": "2026-10-01T16:12:30Z",
    "kwh": 40.2, "currency": "MXN",
    "total_cost": 372.78, "total_cost_status": "FINAL",
    "receipt_url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/receipt",
    "tax_invoice": { "status": "ISSUED", "url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/tax_invoice" },
    "created_at": "2026-10-01T15:30:02Z",
    "updated_at": "2026-10-01T16:12:44Z"
  }
}
```

Errors: `SESSION_NOT_FOUND` 404

### `GET /sessions`

Your sessions.

<details markdown="1">
<summary>Optional parameters</summary>

| Query | Meaning |
| --- | --- |
| `status` | `PENDING`, `ACTIVE`, `COMPLETED`, `FAILED` |
| `updated_since` | Only sessions changed since this time |
| `limit`, `offset` | Paging |

</details>

### `POST /sessions/{id}/stop`

Stop charging. Returns `202` while the charger confirms.

| Error | HTTP |
| --- | --- |
| `SESSION_NOT_FOUND` | 404 |
| `SESSION_NOT_ACTIVE` | 409 |
| `SESSION_NOT_LINKED` (charger has not reported the session yet, retry shortly) | 409 |
| `SESSION_STOP_FAILED` | 502 |

### `PUT /sessions/{id}/retail_total`

Wholesale partners only: the price you charged your driver.

| Body field | Meaning |
| --- | --- |
| `amount` **required** | Incl. VAT |
| `currency` **required** | |

## Charging webhooks and live stream

| Event | When |
| --- | --- |
| `session.started` · `session.updated` · `session.completed` · `session.failed` | Live session events. `session.updated` whenever kWh or cost so far changes. |

```json
{
  "id": "evt_01J9XK2",
  "type": "session.completed",
  "created_at": "2026-10-01T16:12:44Z",
  "data": {
    "id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "status": "COMPLETED",
    "location_id": "LOC-CDMX-0142",
    "charge_point_id": "MX*EXA*E0142A",
    "connector_id": "1",
    "start_time": "2026-10-01T15:30:41Z", "end_time": "2026-10-01T16:12:30Z",
    "kwh": 40.2, "currency": "MXN",
    "total_cost": 372.78, "total_cost_status": "FINAL",
    "receipt_url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/receipt",
    "tax_invoice": { "status": "ISSUED", "url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/tax_invoice" },
    "created_at": "2026-10-01T15:30:02Z",
    "updated_at": "2026-10-01T16:12:44Z"
  }
}
```

### `GET /events`

Optional server-sent events stream with the same events, for live in-app screens.

<details class="section" id="receipts-and-tax-invoices" markdown="1">
<summary>Receipts and tax invoices</summary>

Every public price session gets a receipt automatically. The tax invoice is issued automatically
once you send the driver's billing details, at session start or afterwards. For wholesale sessions
you are the seller, so you issue your own.

### `GET /sessions/{id}/receipt`

Returns the receipt PDF.

### `PUT /sessions/{id}/billing_details`

Send or correct the driver's tax data.

| Body field | Meaning |
| --- | --- |
| `tax_id` **required** | e.g. RFC in Mexico |
| `legal_name` **required** | |
| `postal_code` **required** | |
| `tax_regime` **required in Mexico** | |
| `invoice_use` **required in Mexico** | |
| `email` | Where to send the tax invoice |
| `country` | ISO code, e.g. `MX` |

```json
{
  "data": {
    "session_id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "tax_invoice": { "status": "REQUESTED" }
  }
}
```

| Error | HTTP |
| --- | --- |
| `INVALID_BILLING_DETAILS` | 422 |
| `TAX_INVOICE_NOT_AVAILABLE` (wholesale session) | 409 |
| `TAX_INVOICE_ALREADY_ISSUED` | 409 |
| `SESSION_NOT_FOUND` | 404 |

### `GET /sessions/{id}/tax_invoice`

Returns the tax invoice PDF once issued.

| Error | HTTP |
| --- | --- |
| `TAX_INVOICE_NOT_READY` | 409 |
| `SESSION_NOT_FOUND` | 404 |

### Event `tax_invoice.issued`

Carries `reference` and the tax invoice `url`.

</details>

<details class="section" id="disputes" markdown="1">
<summary>Disputes</summary>

### `POST /sessions/{id}/disputes`

Report a refund, chargeback or wrong charge on a session.

| Body field | Meaning |
| --- | --- |
| `type` **required** | `REFUND`, `CHARGEBACK`, `BILLING_ERROR` |
| `amount` **required** | Disputed amount |
| `reason` | |

```json
{
  "data": {
    "id": "dsp_31Kc",
    "session_id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "type": "CHARGEBACK",
    "amount": 120.00,
    "currency": "MXN",
    "status": "UNDER_REVIEW"
  }
}
```

### `GET /disputes/{id}`

One dispute. `status` is `UNDER_REVIEW`, `ACCEPTED` or `REJECTED`. Once decided, the response
carries the outcome:

```json
{
  "data": {
    "id": "dsp_31Kc",
    "session_id": "ses_7Hq2Lx",
    "reference": "ORDER-20261001-000231",
    "type": "CHARGEBACK",
    "amount": 120.00,
    "currency": "MXN",
    "status": "ACCEPTED",
    "outcome": { "amount": 120.00, "currency": "MXN", "effect": "SET_OFF_NEXT_SETTLEMENT" }
  }
}
```

Errors: `DISPUTE_NOT_FOUND` 404

### `GET /sessions/{id}/evidence`

Returns the evidence PDF for a chargeback: session timeline, meter values, the charging partner's charge
record and our checks.

### Event `dispute.updated`

Carries `reference`, `status` and `outcome`.

</details>

<details class="section" id="payments" markdown="1">
<summary>Payments (optional)</summary>

Tell us which sessions you collected from drivers, so we can match your payment to them. Needs a
write key.

### `POST /payments`

| Body field | Meaning |
| --- | --- |
| `reference` **required** | Your own ID for this payment, up to 200 characters |
| `paid_at` **required** | When you collected it, ISO 8601 |
| `currency` **required** | ISO 4217, e.g. `MXN` |
| `sessions` **required** | 1 to 100 items, each with `session_id` and `amount` |

```json
{
  "reference": "PAY-2026-10-03-001",
  "paid_at": "2026-10-03T09:00:00Z",
  "currency": "MXN",
  "sessions": [{ "session_id": "ses_7Hq2Lx", "amount": 372.78 }]
}
```

Returns `201` with the stored payment and `received_at`. Sending the same `reference` and body again
returns `200` with what was stored.

| Error | HTTP |
| --- | --- |
| `INVALID_PARAMETER` | 422 |
| `INVALID_SESSIONS` (a session is unknown or in another currency; nothing is stored) | 422 |
| `REFERENCE_REUSED` (same `reference`, different body) | 409 |

</details>
