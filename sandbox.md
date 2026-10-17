# Sandbox

Test your integration before going live. Simulated stations, no real energy, no real money.

| | Sandbox | Production |
|---|---|---|
| Base URL | `https://dp.networkcore.org` | `https://dp.networkcore.org` |
| Keys | `nc_test_` keys | `nc_live_` keys |

Sandbox and production keys are separate. Ask your NetworkCore contact for yours.
Sandbox data can be reset at any time.

---

## Distribution Partners

Each test charge point is its own location, `NC-` plus its name, for example `NC-SBX-OK`. Each one behaves the same way every time.

| Charge point | What happens | What you get |
|---|---|---|
| `SBX-OK` | Normal charge. About 40 kWh in 30 minutes, stops when you call stop. | `PENDING` → `ACTIVE` → `COMPLETED` with `kwh` and `total_cost` |
| `SBX-REJECT` | Station refuses to start. | `201 PENDING`, then `FAILED` |
| `SBX-BUSY` | Connector already in use. | `409 CHARGE_POINT_OCCUPIED` |
| `SBX-TOKEN` | Driver can't be authorized at the charging partner. | `502 TOKEN_ERROR` |
| `SBX-DROP` | Driver unplugs after 1 minute. | `ACTIVE` → `COMPLETED` without a stop call |
| `SBX-LATE` | Final cost arrives 10 minutes after the session ends. | `COMPLETED` with `total_cost_status: ESTIMATED`, then `FINAL` |
| `SBX-HOLD` | Final cost is held for review. | `COMPLETED` with `total_cost: null` and `total_cost_status: IN_REVIEW` |
| `SBX-NOTARIFF` | No tariff available at start. | `422 NO_TARIFF` |
| `SBX-MX` | Normal charge in Mexico City, priced in MXN. | Same as `SBX-OK` |

**Also test on any charge point**

- Sandbox sessions are simulated: there are no webhooks, so read them with `GET /sessions/{id}`.
- Send the same start request twice. The second one returns the original response with `Idempotent-Replayed: true`.
- Reuse a reference with a different charge point. You get `422 DUPLICATE_REFERENCE`.

Errors always look like this:

    { "error": { "code": "CHARGE_POINT_OCCUPIED", "message": "..." } }

**Before going live**

1. Run every test station above and handle each result in your app.
2. Share a test run with your NetworkCore contact.
3. Switch to production keys and base URL.

---

## Charge Point Operators

Connect your OCPI 2.2.1 test environment to our sandbox. We run the full flow against your CSMS.

**What we test**

1. Credentials exchange (`versions` and `details`).
2. Locations and tariffs sync.
3. Token push and authorization.
4. `START_SESSION` and `STOP_SESSION` on a charge point you choose.
5. Session updates while charging.
6. CDR delivery, checked against your tariff.

**Before going live**

1. All six steps pass.
2. We confirm the CDR totals match your tariff.
3. Switch to production credentials.
