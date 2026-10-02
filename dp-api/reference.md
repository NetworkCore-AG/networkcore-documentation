# DP API reference

One API for distribution partners such as mobility apps, fleets and car makers: find chargers,
show live availability and prices, start and stop charging, follow every session live.

| | |
| --- | --- |
| Base URL | `https://dp.networkcore.org` |
| Auth | `Authorization: Bearer <api_key>` |
| Format | JSON, amounts incl. VAT, ISO 8601 UTC |

## Integration levels

- **Data**: locations, live availability and prices. No money moves and no sessions start. Fits
  maps, POI display and route planners.
- **Full charging**: everything in Data, plus starting and stopping sessions and live session
  events.

| Group | Data | Full charging |
| --- | --- | --- |
| [Locations and availability](#locations-and-availability) | ✓ | ✓ |
| [Prices](#prices) | ✓ | ✓ |
| [Data webhooks](#data-webhooks) | ✓ | ✓ |
| [Sessions](#sessions) | | ✓ |
| [Charging webhooks and live stream](#charging-webhooks-and-live-stream) | | ✓ |
| [Receipts and tax invoices](#receipts-and-tax-invoices) | | ✓ |
| [Disputes](#disputes) | | ✓ |

## Conventions

### Your order ID

Every session carries `order_id`: your own order or transaction ID. It is required, unique per
partner, and comes back on every session, webhook, tax invoice and dispute. Sending the same
`order_id` twice replays the first answer instead of starting a second charge.

### Errors

One shape everywhere:

```json
{ "error": { "code": "CHARGE_POINT_OCCUPIED",
             "message": "Charge point EVSE-01 already has an active session." } }
```

### Headers

- `X-Correlation-ID` on every response. Quote it when you report a problem.
- `Idempotent-Replayed: true` when an answer is a replay of an earlier request.

### Lists and syncing

- `limit` and `offset`, with `total_count` in the response.
- `updated_since` returns only what changed, for cheap delta syncs.

### Webhooks

**Setup.** Register one HTTPS endpoint in the NetworkCore portal. You get a signing secret (starts
with `whsec_`), shown once. All events go to that endpoint.

**Payload.** Every event has `id`, `type`, `created_at` and `data`:

```json
{
  "id": "evt_01J9XK2",
  "type": "session.completed",
  "created_at": "2026-10-01T16:12:44Z",
  "data": {
    "id": "ses_7Hq2Lx", "order_id": "ORDER-20261001-000231",
    "status": "COMPLETED", "kwh": 40.2,
    "total_cost": 372.78, "currency": "MXN",
    "receipt_url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/receipt",
    "tax_invoice": { "status": "ISSUED", "url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/tax_invoice" }
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
- Secret rotation: when you rotate in the portal, old and new secrets both work for 24 hours.

### Settlement and reconciliation

Every session carries your `order_id` and its final amounts, so you can match it in your own
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

Chargers near a point or inside a map area, with live status.

<details markdown="1">
<summary>Optional parameters</summary>

| Query | Meaning |
| --- | --- |
| `lat`, `lng`, `radius_km` | Search around a point |
| `bbox` | Or a map area: `min_lng,min_lat,max_lng,max_lat` |
| `country` | ISO code, e.g. `MX` |
| `min_power_kw` | e.g. `50` for fast charging |
| `connector` | `CCS2`, `TYPE2`, `CHADEMO`, `GBT`… |
| `available_only` | `true` to hide busy or broken chargers |
| `updated_since` | Only locations changed since this time |
| `limit`, `offset` | Paging |

</details>

```json
{
  "data": [{
    "id": "LOC-CDMX-0142",
    "name": "Example Plaza, Level 2",
    "operator": { "id": "MX*EXA", "name": "Example CPO" },
    "address": "Av. Ejemplo 100, Ciudad de México",
    "coordinates": { "lat": 19.4402, "lng": -99.2038 },
    "country": "MX", "time_zone": "America/Mexico_City",
    "charge_points": [{
      "id": "MX*EXA*E0142A",
      "status": "AVAILABLE",
      "connectors": [{ "id": "1", "standard": "CCS2", "power_kw": 120, "tariff_id": "EXA-DC-PUB" }]
    }],
    "last_updated": "2026-10-01T15:04:11Z"
  }],
  "total_count": 1, "limit": 50, "offset": 0
}
```

### `GET /locations/{id}`

One location with all its charge points, connectors, status and tariffs. Same shape as an item in
`GET /locations`.

Errors: `LOCATION_NOT_FOUND`

### `GET /availability`

Light status feed for keeping a map fresh without re-downloading locations.

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

Every CPO tariff on the network, as structured price components.

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
| `charge_point_id` **required** | |
| `connector_id` **required** | |

<details markdown="1">
<summary>Optional fields</summary>

| Body field | Meaning |
| --- | --- |
| `estimate.kwh` | Expected energy, e.g. `40` |
| `estimate.minutes` | Expected duration, for time-based prices |

</details>

```json
{
  "data": {
    "currency": "MXN",
    "pricing_mode": "PUBLIC",
    "total_incl_vat": 371.00,
    "vat": 51.17,
    "breakdown": [
      { "type": "ENERGY", "quantity": 40, "unit": "kWh", "amount": 356.00 },
      { "type": "FLAT",   "quantity": 1,  "unit": "session", "amount": 15.00 }
    ],
    "valid_until": "2026-10-01T16:00:00Z"
  }
}
```

## Data webhooks

| Event | When |
| --- | --- |
| `evse.status_changed` | A charge point became available, busy or out of order |
| `location.updated` | A location was added, changed or removed |
| `tariff.updated` | A CPO changed a price |

---

## Sessions

### `POST /sessions`

Start charging. We send the start command to the charger.

| Body field | Meaning |
| --- | --- |
| `order_id` **required** | Your order ID. Unique per partner. Same ID = same answer. |
| `location_id` **required** | |
| `charge_point_id` **required** | OCPI EVSE uid |
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
    "order_id": "ORDER-20261001-000231",
    "status": "PENDING",
    "location_id": "LOC-CDMX-0142",
    "charge_point_id": "MX*EXA*E0142A",
    "connector_id": "1",
    "start_time": null, "end_time": null,
    "kwh": 0, "currency": "MXN", "total_cost": null,
    "created_at": "2026-10-01T15:30:02Z",
    "updated_at": "2026-10-01T15:30:02Z"
  }
}
```

| Error | HTTP |
| --- | --- |
| `MISSING_FIELD` | 422 |
| `LOCATION_NOT_FOUND` | 404 |
| `CHARGE_POINT_NOT_FOUND` | 404 |
| `CONNECTOR_NOT_FOUND` | 404 |
| `CHARGE_POINT_OCCUPIED` | 409 |
| `REFERENCE_CONFLICT` | 409 |
| `REQUEST_IN_PROGRESS` | 409 |
| `SESSION_START_FAILED` | 502 |

### `GET /sessions/{id}`

One session, live: energy so far, cost so far, status.

```json
{
  "data": {
    "id": "ses_7Hq2Lx",
    "order_id": "ORDER-20261001-000231",
    "status": "COMPLETED",
    "location_id": "LOC-CDMX-0142",
    "charge_point_id": "MX*EXA*E0142A",
    "connector_id": "1",
    "start_time": "2026-10-01T15:30:41Z", "end_time": "2026-10-01T16:12:30Z",
    "kwh": 40.2, "currency": "MXN", "total_cost": 372.78,
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
| `order_id` | Look up a session by your order ID |
| `limit`, `offset` | Paging |

</details>

### `POST /sessions/{id}/stop`

Stop charging. Returns `202` while the charger confirms.

| Error | HTTP |
| --- | --- |
| `SESSION_NOT_FOUND` | 404 |
| `SESSION_NOT_ACTIVE` | 409 |
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
| `session.started` · `session.updated` · `session.completed` · `session.failed` | Live session events. `session.updated` every few minutes with kWh and cost so far. |

```json
{
  "id": "evt_01J9XK2",
  "type": "session.completed",
  "created_at": "2026-10-01T16:12:44Z",
  "data": {
    "id": "ses_7Hq2Lx", "order_id": "ORDER-20261001-000231",
    "status": "COMPLETED", "kwh": 40.2,
    "total_cost": 372.78, "currency": "MXN",
    "receipt_url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/receipt",
    "tax_invoice": { "status": "ISSUED", "url": "https://dp.networkcore.org/sessions/ses_7Hq2Lx/tax_invoice" }
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
    "order_id": "ORDER-20261001-000231",
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

Carries `order_id` and the tax invoice `url`.

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
    "order_id": "ORDER-20261001-000231",
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
    "order_id": "ORDER-20261001-000231",
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

Returns the evidence PDF for a chargeback: session timeline, meter values, the operator's charge
record and our checks.

### Event `dispute.updated`

Carries `order_id`, `status` and `outcome`.

</details>
