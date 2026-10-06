# Payment Service — Integration Guide

You create an **order**, redirect the payer to our hosted checkout, then confirm
the result with `GET /orders/{id}` and fulfil.

One order can have many **payments** (one row per attempt). You fulfil on the
order, keyed by your own `receipt`.

---

## Quick start

1. `POST /api/v1/orders` with your `receipt` and `amount` in paise.
2. Redirect the payer's browser to `checkout_url` from the response.
3. The payer comes back to your `return_url` with signed query parameters.
   Verify the signature.
4. Call `GET /api/v1/orders/{order_id}`. If `status` is `paid`, fulfil.
5. Run a scheduled sweep for orders that never came back.

If we have configured a webhook URL for you, you also get a signed `POST` on
every order status change. It carries the same fields as the redirect, and you
handle it the same way: verify, poll, fulfil. Without a webhook URL, no webhooks
are sent and steps 1–5 are the whole integration.

---

## Authentication

Send your API key on every request:

```
Authorization: Bearer pay_live_7f3a2b9c_JQ4kR2mXv9pLnB6sT0wYcZaHdEfGiKlMnOpQrStU
```

`X-Api-Key: <token>` also works.

The key is shown once at issue time; we store only its hash. Lost keys are
rotated, not recovered.

---

## 1. Create an order

```http
POST /api/v1/orders
Authorization: Bearer <key>
Content-Type: application/json
```

```json
{
  "receipt": "ORD-2026-0091",
  "amount": 250000,
  "currency": "INR",
  "description": "Seed Fund application fee",
  "customer": { "name": "Asha", "email": "asha@example.in", "phone": "+919000000000" },
  "return_url": "https://portal.example.in/payments/return",
  "cancel_url": "https://portal.example.in/cart",
  "notes": { "application_id": "APP-77" },
  "expires_in": 1800
}
```

| field | required | notes |
|---|---|---|
| `receipt` | yes | Your identifier. Must be unique per client. Acts as the idempotency key. |
| `amount` | yes | Integer paise. `250000` = ₹2,500.00. `250000.00` and `"2.5e5"` are rejected. |
| `currency` | yes | `INR`. |
| `description` | no | Shown on checkout. |
| `customer` | no | `name`, `email`, `phone` — prefills checkout. |
| `return_url` | yes | Must match an origin on your allowlist (scheme, host, port). Ask us to add origins. |
| `cancel_url` | no | Defaults to `return_url`. Same allowlist rule. |
| `notes` | no | Up to 13 string key/value pairs, values max 256 characters. |
| `expires_in` | no | Seconds, 300–86400. Default 1800. |

`201 Created`:

```json
{
  "id": "order_01JQ8ZK3M4N5P6Q7R8S9T0",
  "entity": "order",
  "receipt": "ORD-2026-0091",
  "amount": 250000,
  "amount_paid": 0,
  "amount_due": 250000,
  "currency": "INR",
  "status": "created",
  "attempts": 0,
  "notes": { "application_id": "APP-77" },
  "checkout_url": "https://pay.example.in/pay/9f3c...",
  "expires_at": "2026-08-28T09:30:00Z"
}
```

Redirect the payer to `checkout_url`.

### Retrying create

- Same `receipt`, same amount and currency → `200` with the original order. Safe to retry; no double charge.
- Same `receipt`, different amount → `409`. Use a new receipt or fix the amount.

### Notes

Your `notes` are stored on the order, forwarded to Razorpay on both the order and
the payment, and returned unchanged by `GET /orders/{id}`.

**Payment notes are visible to the payer** — they appear in the checkout page
source. Put nothing secret in them.

We add reserved keys:

| key | on the order | on the payment |
|---|---|---|
| `pay_receipt` | yes | yes |
| `pay_client_id` | yes | no |
| `pay_order_id` | no | yes |

Sending `pay_receipt`, `pay_client_id` or `pay_order_id` yourself is a `422`.

---

## 2. Read order status

```http
GET /api/v1/orders/{order_id}
GET /api/v1/orders/{order_id}/payments
GET /api/v1/payments/{payment_id}
GET /api/v1/orders
```

`amount_paid`, `amount_due` and `attempts` are computed from the payment rows.

### Order status

| status | meaning |
|---|---|
| `created` | No payment attempted yet. |
| `attempted` | At least one payment tried, none succeeded. |
| `paid` | Terminal. Fulfil on this. |
| `expired` | Checkout window elapsed. |
| `cancelled` | Cancelled before payment. |

**`expired` and `cancelled` are not final.** A payment already in flight when the
window closes can still be captured, so an order can go `expired` and then `paid`
— usually within a minute, sometimes after a reconciliation sweep. Do not tear
down or refund on `expired`. Only `paid` is final.

### Payment status

`created` → `authorized` → `captured`, or `failed`. A retry after a failure
creates a new payment row.

---

## 3. Verify the signature

The redirect and the webhook carry the same four signed fields:

| field | example |
|---|---|
| `order_id` | `order_01JQ8ZK3M4N5P6Q7R8S9T0VWX` |
| `receipt` | `ORD-2026-0091` |
| `status` | the order's status |
| `ts` | unix timestamp of the status change |

The signature is a hex HMAC-SHA256 of the four values joined with `|` in exactly
this order, using your signing secret:

```
order_id|receipt|status|ts
```

`ts` is joined as a plain decimal integer — no quotes, no padding.

It arrives as the `signature` query parameter on the redirect, and as the
`X-Pay-Signature` header on the webhook.

```php
function payVerify(array $f, string $signature, string $secret): bool
{
    $canonical = implode('|', [$f['order_id'], $f['receipt'], $f['status'], $f['ts']]);

    return hash_equals(hash_hmac('sha256', $canonical, $secret), $signature);
}
```

- Redirect: `payVerify($_GET, $_GET['signature'], $secret)`
- Webhook: `payVerify($request->json()->all(), $request->header('X-Pay-Signature'), $secret)`

One function covers both.

<details>
<summary><strong>Node.js</strong></summary>

```js
const crypto = require('crypto');

function verifyPaySignal(fields, signature, secret, { toleranceSeconds = 300 } = {}) {
  const { order_id, receipt, status, ts } = fields || {};

  if (!order_id || !receipt || !status || !ts || !signature) return false;

  const canonical = [order_id, receipt, status, String(ts)].join('|');

  const expected = crypto.createHmac('sha256', secret).update(canonical, 'utf8').digest('hex');

  const a = Buffer.from(expected, 'utf8');
  const b = Buffer.from(String(signature), 'utf8');

  // timingSafeEqual throws on length mismatch, so check length first.
  if (a.length !== b.length) return false;
  if (!crypto.timingSafeEqual(a, b)) return false;

  const age = Math.abs(Math.floor(Date.now() / 1000) - Number(ts));

  return Number.isFinite(age) && age <= toleranceSeconds;
}
```

Webhook — no raw-body middleware needed; the canonical string is rebuilt from the
parsed fields:

```js
app.post('/webhooks/pay', express.json(), async (req, res) => {
  if (!verifyPaySignal(req.body, req.get('X-Pay-Signature'), process.env.PAY_SIGNING_SECRET)) {
    return res.sendStatus(400);
  }

  const { order_id, status } = req.body;

  if (await alreadyHandled(order_id, status)) return res.sendStatus(204);

  res.sendStatus(200);          // ack fast, then do the work
  await queue.add('pay:check', { order_id });
});
```

Redirect — same function, fields off the query string:

```js
app.get('/payments/return', async (req, res) => {
  if (!verifyPaySignal(req.query, req.query.signature, process.env.PAY_SIGNING_SECRET)) {
    return res.status(400).send('Invalid signature');
  }

  const order = await payApi.get(`/orders/${req.query.order_id}`);

  res.render(order.status === 'paid' ? 'thanks' : 'payment-pending');
});
```

</details>

---

## 4. Poll, then fulfil

Never fulfil on the redirect or webhook payload. Verify it, then read the order:

```php
if (! payVerify($fields, $signature, $signingSecret)) {
    abort(400);
}

$order = $http->get("/api/v1/orders/{$fields['order_id']}")->json();

if ($order['status'] === 'paid' && $order['amount_paid'] >= $order['amount']) {
    fulfil($order['receipt']);   // must be idempotent
}
```

Use the `status` you were sent only to pick which page to show. Reject anything
whose `ts` is older than five minutes.

---

## 5. Sweep open orders

Payers close tabs. On a schedule, re-check every order you created that is still
`created` or `attempted` after a few minutes, until it reaches a terminal status.
Without this you will miss payments where the payer never returned.

---

## Webhooks

Only sent if we have configured a webhook URL for you.

### Events

One webhook per order status change:

| `X-Pay-Event` | when |
|---|---|
| `order.attempted` | a payment was tried and did not succeed |
| `order.paid` | fulfil on this, after polling |
| `order.expired` | checkout window elapsed — not final |
| `order.cancelled` | cancelled before payment |

Individual payment attempts are not signalled — two declines on one order produce
one `order.attempted`. For attempt-level detail (decline reason, method) read
`GET /orders/{id}/payments`.

### Request shape

```http
POST /your/webhook/endpoint
Content-Type: application/json
X-Pay-Event-Id: 9f3c1e2a-7b44-4c81-9d2e-5a6b0c3d8e17
X-Pay-Event: order.paid
X-Pay-Signature: cd660a9c9564503269be2a7fdf56432e569b722202579883742ec683273b6da0

{
  "order_id": "order_01JQ8ZK3M4N5P6Q7R8S9T0VWX",
  "receipt": "ORD-2026-0091",
  "status": "paid",
  "ts": 1774689000
}
```

`X-Pay-Event` and `X-Pay-Event-Id` are headers, outside the signature. Read the
real status from the body.

### Handling

1. Verify with `payVerify()`.
2. Deduplicate on **order id + status** — both are signed. `X-Pay-Event-Id` is
   unsigned; quote it to us when reporting a delivery, but do not use it as your
   only idempotency key.
3. Return `2xx` immediately and queue the work. We time out after 20 seconds.

```php
if ($alreadyHandled($body['order_id'], $body['status'])) {
    return response()->noContent();
}
```

### Retries

Anything that is not `2xx` — 3xx, 4xx, 5xx, timeout — is retried up to 8 times:

`10s → 30s → 2m → 10m → 30m → 2h → 6h → 12h` (±10% jitter), then dead-lettered.

Return `410 Gone` to stop delivery of that event permanently. Dead-lettered
deliveries can be redriven by hand — tell us if you had an outage.

Keep the sweep from step 5 even with webhooks. If `order.paid` never arrived for
an order you expected to complete, poll it and tell us.

---

## Errors

| status | meaning | retry? |
|---|---|---|
| `401` | Missing, malformed, unknown, revoked or expired key, or suspended client. | no |
| `404` | No such order or payment for you. | no |
| `409` | `receipt` reused with different terms. | no — fix the amount or use a new receipt |
| `422` | Validation failed. | no |
| `429` | Rate limited — 60 order creations and 300 reads per minute. | yes, after backoff |
| `502` | We could not reach Razorpay. | yes — reuse the same `receipt` |

`401`, `429` and `502`:

```json
{ "error": { "code": "gateway_error", "message": "..." } }
```

`422` uses the Laravel validation shape:

```json
{ "message": "The amount must be an integer number of paise.",
  "errors": { "amount": ["The amount must be an integer number of paise."] } }
```

---

## Checklist before going live

- [ ] Every fulfilment path calls `GET /orders/{id}` and reads the status from that response.
- [ ] Fulfilment is idempotent, keyed on `receipt` or on order id + status.
- [ ] `expired` and `cancelled` are not treated as final.
- [ ] Signature verification rebuilds `order_id|receipt|status|ts` in that order and uses a constant-time compare.
- [ ] One verifier for both redirect and webhook.
- [ ] Amounts are integer paise everywhere.
- [ ] Return and cancel origins are on the allowlist.
- [ ] Webhook endpoint returns 2xx fast and queues the work.
- [ ] A sweep exists for orders that never reached a terminal status.

---

## Building with an AI coding assistant

A packaged agent skill covering this whole integration lives in
[`skills/pay-integration/`](skills/pay-integration/SKILL.md). Copy it into your
portal repository so your assistant follows the contract instead of guessing at
it — see [`skills/README.md`](skills/README.md) for install instructions.
