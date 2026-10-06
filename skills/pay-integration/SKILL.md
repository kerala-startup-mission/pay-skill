---
name: pay-integration
description: Integrate a client portal with the Pay payment service (hosted Razorpay checkout) — create orders, redirect to hosted checkout, verify the HMAC-signed return redirect and webhook, poll GET /orders/{id}, fulfil idempotently, and sweep stale orders. Use whenever the task involves taking a payment, a checkout flow, an order/receipt/paise amount, a `X-Pay-Signature` or `signature` query parameter, a `pay_live_`/`pay_test_` API key, or a `/api/v1/orders` call.
---

# Pay integration (client portal side)

You are implementing the **client portal** side. The Pay service is already
built; do not modify it, do not call Razorpay directly, and do not reimplement
checkout UI. The portal only does five things:

1. `POST /api/v1/orders` → get `checkout_url`
2. Redirect the payer's browser to `checkout_url`
3. Payer returns to `return_url` with signed query parameters → **verify signature**
4. `GET /api/v1/orders/{id}` → if `status == "paid"`, **fulfil idempotently**
5. A scheduled sweep re-polls non-terminal orders

Webhooks are optional and additive. They arrive only if Pay has a webhook URL
configured for this client. They do the same thing as step 3: verify, poll,
fulfil.

## The one rule that governs everything

**Never fulfil on the redirect or the webhook payload.** Both are signals, not
facts. A payer can replay an old valid success URL. The only authority is the
response body of `GET /api/v1/orders/{id}`.

Signal tells you *which page to render*. `GET /orders/{id}` tells you *whether
to ship the goods*.

## Before writing code

Confirm these exist in the portal's config/secrets (never hardcode, never commit):

| env var | what it is |
|---|---|
| `PAY_BASE_URL` | e.g. `https://pay.example.in` |
| `PAY_API_KEY` | `pay_live_...` / `pay_test_...`, sent as `Authorization: Bearer <key>` |
| `PAY_SIGNING_SECRET` | HMAC-SHA256 secret, **different from the API key** |

Also confirm the portal's `return_url` origin is on the Pay allowlist. If it is
not, order creation fails `422` on `return_url`. The portal cannot self-serve
this — tell the user to ask the Pay operator to add the origin (scheme, host and
port must match exactly).

## Implementation order

Build in this order; each step is independently testable.

### 1. A thin API client

One place that sets base URL, `Authorization: Bearer`, `Accept:
application/json`, a timeout (10s is sane), and maps non-2xx to typed errors.
Everything else calls through it. See `references/api-reference.md` for every
endpoint, field and error code.

### 2. Order creation

```
POST /api/v1/orders
{ "receipt": "...", "amount": 250000, "currency": "INR",
  "return_url": "https://portal.example.in/payments/return" }
```

Hard requirements:

- **`amount` is integer paise.** `250000` = ₹2,500.00. Never build it with
  floats. `round($rupees * 100)` on a float is a bug waiting for `19.99`; use
  integer/decimal arithmetic or store paise natively. `250000.00` and `"2.5e5"`
  are rejected `422`.
- **`receipt` is your own identifier and is the idempotency key.** It must be
  unique per client and match `^[A-Za-z0-9._-]+$`, max 120 chars. Derive it from
  a row in your own database (`ORD-2026-0091`, `APP-77-FEE`) — never from a
  timestamp or random value, or a retry creates a second order.
- Persist `id`, `receipt`, `status`, `amount` and `expires_at`
  in the portal's own DB **before** redirecting. If the redirect is the only
  record, an abandoned payment is unrecoverable.

Retry semantics you must honour:

- Same `receipt`, same amount + currency → `200` (not `201`) with the original
  order. Safe to retry. No double charge. Treat 200 and 201 identically.
- Same `receipt`, different amount → `409`. Do not retry — this is a bug in the
  portal (amount changed under a reused receipt). Allocate a new receipt.
- `502` → Pay could not reach Razorpay. Retry **with the same receipt**.

Then `redirect($order['checkout_url'])` — a full browser navigation, not fetch
or XHR.

### 3. The verifier — one function, both channels

The redirect and the webhook carry the **same four fields** and the **same
canonical string**:

```
order_id|receipt|status|ts
order_01JQ8ZK3M4N5P6Q7R8S9T0VWX|ORD-2026-0091|paid|1774689000
```

`signature` = hex `HMAC-SHA256(canonical, PAY_SIGNING_SECRET)`.

- Redirect: fields are query parameters; signature is the `signature` query parameter.
- Webhook: fields are JSON body keys; signature is the `X-Pay-Signature` header.

Write **one** verifier used by both. Non-negotiable properties:

- Rebuild the canonical string from the received fields in **exactly** that
  order. Do not sort, do not include extra parameters, do not include
  `signature` itself.
- `ts` joins as a plain decimal integer — no quotes, no zero padding.
- Constant-time compare (`hash_equals`, `crypto.timingSafeEqual`,
  `hmac.compare_digest`). Never `==`.
- Reject `ts` older than 300 seconds (replay window).
- Missing any field → reject.

Language-ready implementations: `references/php-laravel.md`, `references/node-express.md`.

### 4. The return handler (`return_url`)

```
verify(query) or 400
order = GET /api/v1/orders/{order_id}
if order.status == "paid" and order.amount_paid >= order.amount: fulfil(order.receipt)
render(order.status == "paid" ? thanks : pending)
```

The route must be **GET**, publicly reachable, and CSRF-exempt if the framework
guards GET routes. It must tolerate being hit twice with the same parameters.

`cancel_url` receives the same signed envelope and is verified the same way. If
you did not send one, cancellations land on `return_url` with a non-`paid`
status — handle that branch.

### 5. Fulfilment

One function, idempotent, keyed on `receipt` (or order id + status). Guard it
with a DB-level unique constraint or a `SELECT ... FOR UPDATE` / transactional
state check — not an `if (!$order->fulfilled)` read outside a transaction. The
return handler, the webhook and the sweep will all call it, sometimes
concurrently.

Verify the amount before fulfilling: `amount_paid >= amount`. Never trust a
client-supplied amount at this point; use the values from the `GET` response.

### 6. Status handling

| status | portal action |
|---|---|
| `created` | keep polling |
| `attempted` | keep polling — a payment failed, payer may retry |
| `paid` | **terminal.** Fulfil. |
| `expired` | **not final.** Keep polling for a while. |
| `cancelled` | **not final.** Keep polling for a while. |

`expired` and `cancelled` can still become `paid` — a payment in flight when the
window closed can still capture, usually within a minute, sometimes after a
reconciliation sweep. **Do not release inventory, void the application, or issue
a refund on `expired`.** Only `paid` is final.

### 7. The sweep (not optional)

A scheduled job — every 5 minutes is fine — that re-polls every order the portal
created that is not yet `paid` and is older than a couple of minutes. Payers
close tabs; without the sweep the portal silently loses payments. Keep the sweep
even after webhooks are enabled.

Stop polling an order after a generous cutoff (24h past `expires_at` is
reasonable) and flag it for a human rather than deleting it.

### 8. Webhooks (only if a webhook URL is configured)

```
POST /your/webhook/endpoint
X-Pay-Event: order.paid
X-Pay-Event-Id: 9f3c1e2a-...
X-Pay-Signature: cd660a9c...

{"order_id":"order_...","receipt":"ORD-2026-0091","status":"paid","ts":1774689000}
```

- Verify with the **same** verifier. No raw-body middleware needed — the
  canonical string is rebuilt from parsed fields, not from the raw bytes.
- Read status from the **body**, not from `X-Pay-Event`. The headers are outside
  the signature.
- Deduplicate on **order id + status** (both signed). `X-Pay-Event-Id` is
  unsigned — log it for support, never use it as the only idempotency key.
- Return `2xx` immediately, then queue the poll + fulfil. Pay times out at 20
  seconds.
- Events: `order.attempted`, `order.paid`, `order.expired`, `order.cancelled`.
  Individual payment attempts are not signalled — two declines produce one
  `order.attempted`. For decline reasons read `GET /orders/{id}/payments`.
- Non-2xx (including 3xx) is retried 8 times: `10s → 30s → 2m → 10m → 30m → 2h →
  6h → 12h` (±10% jitter), then dead-lettered. Return `410 Gone` only to stop
  that event permanently.
- The endpoint must be CSRF-exempt.

## Anti-patterns — reject these if you see them in existing portal code

- Fulfilling from the redirect's `status=paid` without calling `GET /orders/{id}`.
- Verifying with `==` / `===` instead of a constant-time compare.
- Signing or verifying a JSON blob or a sorted parameter list instead of the
  fixed `order_id|receipt|status|ts` join.
- No `ts` freshness check.
- Floats anywhere near `amount`.
- Random or timestamp-derived `receipt`.
- Treating `expired`/`cancelled` as final (releasing seats, cancelling the application).
- Webhook handler doing the fulfilment work inline before responding.
- The API key doubling as the signing secret.
- Logging the API key, the signing secret, or a full `checkout_url` (its token
  grants access to the payment page).

## Before calling the integration done

- [ ] Every fulfilment path reads status from a `GET /orders/{id}` response.
- [ ] Fulfilment is idempotent under concurrency, with a DB-level guard.
- [ ] One verifier serves both redirect and webhook; constant-time compare; `ts` window enforced.
- [ ] Amounts are integer paise end to end.
- [ ] `receipt` is stable and derived from portal data.
- [ ] `201` and `200` from order creation are handled the same way; `409` is not retried; `502` is retried with the same receipt.
- [ ] `expired`/`cancelled` are not treated as final.
- [ ] Return and cancel origins are on the Pay allowlist.
- [ ] Webhook endpoint (if any) is CSRF-exempt, returns 2xx fast, queues the work, dedupes on order id + status.
- [ ] A scheduled sweep re-polls non-terminal orders.
- [ ] Secrets come from env; nothing sensitive is logged.

## Testing without a live payer

- Order creation, `409`, `422` and the sweep are testable against the API with a
  test key.
- Signature verification is testable offline: build a canonical string, HMAC it
  with a known secret, assert accept; then flip one byte, stale the `ts`, and
  drop a field, and assert reject on each.
- The webhook handler is testable by POSTing a locally-signed body.

Write those tests. A signature verifier with no negative test is a verifier that
does not work.

## References

- `references/api-reference.md` — endpoints, request/response fields, statuses, errors.
- `references/php-laravel.md` — Laravel client, verifier, routes, sweep command.
- `references/node-express.md` — Express client, verifier, routes, sweep job.
