# Pay integration — Node / Express

Adapt names to the portal's conventions. The signature verifier and the
"never fulfil on the signal" rule are not adaptable. The same verifier drops
into Next.js route handlers, Fastify, Nest — only the request plumbing changes.

## Config

```js
// config/pay.js
export const pay = {
  baseUrl: process.env.PAY_BASE_URL.replace(/\/$/, ''),
  apiKey: process.env.PAY_API_KEY,
  signingSecret: process.env.PAY_SIGNING_SECRET,
  toleranceSeconds: Number(process.env.PAY_SIGNATURE_TOLERANCE ?? 300),
};
```

## The verifier — one function, both channels

```js
import crypto from 'node:crypto';

export function verifyPaySignal(fields, signature, secret, { toleranceSeconds = 300 } = {}) {
  const { order_id, receipt, status, ts } = fields || {};

  if (!order_id || !receipt || !status || !ts || !signature) return false;

  // Fixed order. Never sorted, never JSON, never includes `signature`.
  const canonical = [order_id, receipt, status, String(Number(ts))].join('|');

  const expected = crypto.createHmac('sha256', secret).update(canonical, 'utf8').digest('hex');

  const a = Buffer.from(expected, 'utf8');
  const b = Buffer.from(String(signature), 'utf8');

  // timingSafeEqual throws on length mismatch, so compare lengths first.
  if (a.length !== b.length) return false;
  if (!crypto.timingSafeEqual(a, b)) return false;

  const age = Math.abs(Math.floor(Date.now() / 1000) - Number(ts));

  return Number.isFinite(age) && age <= toleranceSeconds;
}
```

## API client

```js
import { pay } from '../config/pay.js';

export class PayError extends Error {
  constructor(status, body) {
    super(body?.message || body?.error?.message || `Pay API error ${status}`);
    this.status = status;
    this.body = body;
  }
}

async function request(method, path, body) {
  const res = await fetch(`${pay.baseUrl}/api/v1${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${pay.apiKey}`,
      Accept: 'application/json',
      ...(body ? { 'Content-Type': 'application/json' } : {}),
    },
    body: body ? JSON.stringify(body) : undefined,
    signal: AbortSignal.timeout(10_000),
  });

  const json = await res.json().catch(() => null);

  if (!res.ok) throw new PayError(res.status, json);

  return json;
}

export const payApi = {
  // 201 create, 200 idempotent replay -- both success, handled identically.
  createOrder: (payload) => request('POST', '/orders', payload),
  getOrder: (id) => request('GET', `/orders/${encodeURIComponent(id)}`),
  getPayments: (id) => request('GET', `/orders/${encodeURIComponent(id)}/payments`),
};
```

Caller handling for the two statuses that mean something specific:

```js
try {
  order = await payApi.createOrder(payload);
} catch (e) {
  if (e.status === 409) throw new Error('Receipt reused with different terms');  // portal bug, do not retry
  if (e.status === 502) return retryLater(payload);                              // same receipt
  throw e;
}
```

## Starting a payment

```js
app.post('/applications/:id/pay', async (req, res) => {
  const application = await Applications.find(req.params.id);

  // Receipt derived from portal data -> a retry hits the same order.
  const receipt = `APP-${application.id}-FEE`;

  const order = await payApi.createOrder({
    receipt,
    amount: application.feePaise,          // integer paise, stored as paise
    currency: 'INR',
    description: 'Application fee',
    customer: { name: application.name, email: application.email, phone: application.phone },
    return_url: `${process.env.APP_URL}/payments/return`,
    cancel_url: `${process.env.APP_URL}/applications/${application.id}`,
    notes: { application_id: String(application.id) },
  });

  // Persist BEFORE redirecting, or an abandoned payment is unrecoverable.
  await PaymentOrders.upsert({
    receipt,
    payOrderId: order.id,
    applicationId: application.id,
    amount: order.amount,
    status: order.status,
    expiresAt: order.expires_at,
  });

  res.redirect(303, order.checkout_url);
});
```

## The return handler

```js
app.get('/payments/return', async (req, res) => {
  if (!verifyPaySignal(req.query, req.query.signature, pay.signingSecret, pay)) {
    return res.status(400).send('Invalid signature');
  }

  // The signal only says "go and check". This call is the authority.
  const order = await payApi.getOrder(req.query.order_id);

  if (order.status === 'paid' && order.amount_paid >= order.amount) {
    await fulfil(order);   // idempotent
  }

  res.render(order.status === 'paid' ? 'thanks' : 'payment-pending', { order });
});
```

## Idempotent fulfilment

```js
export async function fulfil(order) {
  await db.transaction(async (tx) => {
    const row = await tx.paymentOrders.findOne({ receipt: order.receipt }, { forUpdate: true });

    await tx.paymentOrders.update(row.id, {
      status: order.status,
      amountPaid: order.amount_paid,
    });

    if (row.fulfilledAt) return;   // already done -- the row lock makes this safe

    await tx.applications.markFeePaid(row.applicationId);
    await tx.paymentOrders.update(row.id, { fulfilledAt: new Date() });
  });
}
```

Back it with a unique index on `receipt`. The read-then-write check is only safe
inside the transaction holding the row lock.

## Webhook endpoint (only if a webhook URL is configured)

```js
// No raw-body middleware needed: the canonical string is rebuilt from parsed
// fields, not from the raw bytes.
app.post('/webhooks/pay', express.json(), async (req, res) => {
  if (!verifyPaySignal(req.body, req.get('X-Pay-Signature'), pay.signingSecret, pay)) {
    return res.sendStatus(400);
  }

  const { order_id, status } = req.body;   // status from the BODY, not X-Pay-Event

  // Dedupe on order id + status -- both signed. X-Pay-Event-Id is not.
  if (await alreadyHandled(order_id, status)) return res.sendStatus(204);

  res.sendStatus(200);                     // ack fast (20s timeout)
  await queue.add('pay:check', { order_id });
});
```

Exempt this path from CSRF middleware if the portal applies CSRF to POST routes.

## The sweep

```js
// Every 5 minutes.
export async function sweepPayOrders() {
  const rows = await PaymentOrders.find({
    status: { $ne: 'paid' },              // expired/cancelled stay in -- not final
    fulfilledAt: null,
    createdAt: { $gt: hoursAgo(24) },     // give up after a day, flag for a human
    updatedAt: { $lt: minutesAgo(2) },
  }, { limit: 200 });                     // read bucket is 300/min

  for (const row of rows) {
    const order = await payApi.getOrder(row.payOrderId);

    await PaymentOrders.update(row.id, { status: order.status });

    if (order.status === 'paid' && order.amount_paid >= order.amount) {
      await fulfil(order);
    }
  }
}
```

## Next.js route handlers

Same verifier; different plumbing.

```js
// app/payments/return/route.js
export async function GET(request) {
  const fields = Object.fromEntries(new URL(request.url).searchParams);

  if (!verifyPaySignal(fields, fields.signature, process.env.PAY_SIGNING_SECRET)) {
    return new Response('Invalid signature', { status: 400 });
  }

  const order = await payApi.getOrder(fields.order_id);

  if (order.status === 'paid' && order.amount_paid >= order.amount) await fulfil(order);

  return Response.redirect(new URL(order.status === 'paid' ? '/thanks' : '/pending', request.url));
}
```

```js
// app/webhooks/pay/route.js
export async function POST(request) {
  const body = await request.json();

  if (!verifyPaySignal(body, request.headers.get('x-pay-signature'), process.env.PAY_SIGNING_SECRET)) {
    return new Response(null, { status: 400 });
  }

  if (await alreadyHandled(body.order_id, body.status)) return new Response(null, { status: 204 });

  await queue.add('pay:check', { order_id: body.order_id });

  return new Response(null, { status: 200 });
}
```

Both must run on the Node runtime (`export const runtime = 'nodejs'`) —
`node:crypto` is unavailable on the edge runtime.

## Tests worth writing

```js
const sign = (c, s) => crypto.createHmac('sha256', s).update(c).digest('hex');

test('accepts a valid signature', () => {
  const ts = Math.floor(Date.now() / 1000);
  const f = { order_id: 'order_1', receipt: 'R-1', status: 'paid', ts };
  expect(verifyPaySignal(f, sign(`order_1|R-1|paid|${ts}`, 'shh'), 'shh')).toBe(true);
});

test('rejects a tampered status', ...);     // sign 'attempted', send 'paid'
test('rejects a stale ts', ...);            // ts - 600
test('rejects a missing field', ...);       // delete f.receipt
test('return handler does not fulfil when the API says attempted', ...);
```

The last one matters most: stub `getOrder` to return `attempted` while the
redirect query says `paid`, and assert `fulfil` was never called.
