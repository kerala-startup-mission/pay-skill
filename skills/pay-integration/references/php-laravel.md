# Pay integration — PHP / Laravel

Adapt names to the portal's conventions. The signature verifier and the
"never fulfil on the signal" rule are not adaptable.

## config/pay.php

```php
<?php

return [
    'base_url'       => rtrim(env('PAY_BASE_URL', ''), '/'),
    'api_key'        => env('PAY_API_KEY'),
    'signing_secret' => env('PAY_SIGNING_SECRET'),
    'tolerance'      => (int) env('PAY_SIGNATURE_TOLERANCE', 300),
];
```

## The verifier — one function, both channels

```php
<?php

namespace App\Services\Pay;

class PaySignature
{
    /**
     * @param array $fields  Query parameters (redirect) or JSON body (webhook).
     */
    public static function verify(array $fields, ?string $signature, ?int $tolerance = null): bool
    {
        $tolerance ??= (int) config('pay.tolerance');

        foreach (['order_id', 'receipt', 'status', 'ts'] as $key) {
            if (! isset($fields[$key]) || $fields[$key] === '') {
                return false;
            }
        }

        if (! is_string($signature) || $signature === '') {
            return false;
        }

        // Fixed order. Never sorted, never JSON, never includes `signature`.
        $canonical = implode('|', [
            $fields['order_id'],
            $fields['receipt'],
            $fields['status'],
            (string) (int) $fields['ts'],   // plain decimal integer
        ]);

        $expected = hash_hmac('sha256', $canonical, (string) config('pay.signing_secret'));

        if (! hash_equals($expected, $signature)) {
            return false;
        }

        return abs(time() - (int) $fields['ts']) <= $tolerance;
    }
}
```

## API client

```php
<?php

namespace App\Services\Pay;

use Illuminate\Support\Facades\Http;
use Illuminate\Http\Client\PendingRequest;

class PayClient
{
    private function http(): PendingRequest
    {
        return Http::baseUrl(config('pay.base_url').'/api/v1')
            ->withToken(config('pay.api_key'))
            ->acceptJson()
            ->timeout(10)
            ->connectTimeout(5);
    }

    /**
     * 201 on create, 200 on idempotent replay -- both are success.
     *
     * @throws PayConflict   409: receipt reused with different terms (portal bug).
     * @throws PayGatewayDown 502: retry with the SAME receipt.
     */
    public function createOrder(array $payload): array
    {
        $response = $this->http()->post('orders', $payload);

        if ($response->status() === 409) {
            throw new PayConflict($response->json('message') ?? 'Receipt reused with different terms.');
        }

        if ($response->status() === 502) {
            throw new PayGatewayDown($response->json('error.message') ?? 'Gateway unreachable.');
        }

        $response->throw();

        return $response->json();
    }

    public function order(string $orderId): array
    {
        return $this->http()->get("orders/{$orderId}")->throw()->json();
    }

    public function payments(string $orderId): array
    {
        return $this->http()->get("orders/{$orderId}/payments")->throw()->json('items', []);
    }
}
```

## Starting a payment

```php
public function pay(Application $application, PayClient $pay)
{
    // Receipt derived from portal data -> a retry hits the same order.
    $receipt = "APP-{$application->id}-FEE";

    $order = $pay->createOrder([
        'receipt'    => $receipt,
        'amount'     => $application->fee_paise,   // integer paise, stored as paise
        'currency'   => 'INR',
        'description' => 'Application fee',
        'customer'   => [
            'name'  => $application->applicant_name,
            'email' => $application->email,
            'phone' => $application->phone,
        ],
        'return_url' => route('payments.return'),
        'cancel_url' => route('applications.show', $application),
        'notes'      => ['application_id' => (string) $application->id],
    ]);

    // Persist BEFORE redirecting, or an abandoned payment is unrecoverable.
    PaymentOrder::updateOrCreate(
        ['receipt' => $receipt],
        [
            'application_id' => $application->id,
            'pay_order_id'   => $order['id'],
            'amount'         => $order['amount'],
            'status'         => $order['status'],
            'expires_at'     => $order['expires_at'],
        ],
    );

    return redirect()->away($order['checkout_url']);
}
```

## The return handler

```php
// routes/web.php -- GET, public, no auth (the payer may have lost the session).
Route::get('payments/return', [PaymentReturnController::class, 'handle'])->name('payments.return');
```

```php
public function handle(Request $request, PayClient $pay, Fulfilment $fulfilment)
{
    if (! PaySignature::verify($request->query(), $request->query('signature'))) {
        abort(400, 'Invalid payment signature.');
    }

    // The signal only says "go and check". This call is the authority.
    $order = $pay->order($request->query('order_id'));

    if ($order['status'] === 'paid' && $order['amount_paid'] >= $order['amount']) {
        $fulfilment->fulfil($order);   // idempotent
    }

    return view($order['status'] === 'paid' ? 'payments.thanks' : 'payments.pending', [
        'order' => $order,
    ]);
}
```

## Idempotent fulfilment

```php
public function fulfil(array $order): void
{
    DB::transaction(function () use ($order) {
        $row = PaymentOrder::where('receipt', $order['receipt'])->lockForUpdate()->firstOrFail();

        $row->update([
            'status'      => $order['status'],
            'amount_paid' => $order['amount_paid'],
        ]);

        if ($row->fulfilled_at !== null) {
            return;   // already done -- the lock makes this check safe
        }

        $row->application->markFeePaid();
        $row->update(['fulfilled_at' => now()]);
    });
}
```

Back it with a unique index on `receipt`. The read-then-write check is only safe
inside the transaction that holds the row lock.

## Webhook endpoint (only if a webhook URL is configured)

```php
// routes/api.php -- the api group has no CSRF middleware.
Route::post('webhooks/pay', [PayWebhookController::class, 'handle']);
```

```php
public function handle(Request $request)
{
    $body = $request->json()->all();

    if (! PaySignature::verify($body, $request->header('X-Pay-Signature'))) {
        return response()->json(['error' => 'invalid signature'], 400);
    }

    // Dedupe on order id + status -- both signed. X-Pay-Event-Id is not.
    $key = "pay:handled:{$body['order_id']}:{$body['status']}";

    if (! Cache::add($key, 1, now()->addDay())) {
        return response()->noContent();
    }

    // Ack fast (20s timeout), do the work off the request.
    ReconcilePayOrder::dispatch($body['order_id']);

    return response()->noContent();
}
```

If the portal is on the `web` middleware group instead, add the path to the CSRF
exception list in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: ['webhooks/pay']);
})
```

## The sweep

```php
class SweepPayOrders extends Command
{
    protected $signature = 'pay:sweep';

    public function handle(PayClient $pay, Fulfilment $fulfilment): void
    {
        PaymentOrder::whereNotIn('status', ['paid'])
            ->whereNull('fulfilled_at')
            ->where('created_at', '>', now()->subDay())      // give up after a day
            ->where('updated_at', '<', now()->subMinutes(2))
            ->limit(200)                                      // read bucket is 300/min
            ->each(function (PaymentOrder $row) use ($pay, $fulfilment) {
                $order = $pay->order($row->pay_order_id);

                $row->update(['status' => $order['status']]);

                if ($order['status'] === 'paid' && $order['amount_paid'] >= $order['amount']) {
                    $fulfilment->fulfil($order);
                }
            });
    }
}
```

```php
// routes/console.php
Schedule::command('pay:sweep')->everyFiveMinutes()->withoutOverlapping();
```

Note `whereNotIn('status', ['paid'])` — `expired` and `cancelled` stay in the
sweep, because they are not final.

## Tests worth writing

```php
public function test_valid_signature_is_accepted(): void
{
    config(['pay.signing_secret' => 'shh']);

    $fields = ['order_id' => 'order_1', 'receipt' => 'R-1', 'status' => 'paid', 'ts' => time()];
    $signature = hash_hmac('sha256', 'order_1|R-1|paid|'.$fields['ts'], 'shh');

    $this->assertTrue(PaySignature::verify($fields, $signature));
}

public function test_tampered_status_is_rejected(): void   // flip status after signing
public function test_stale_timestamp_is_rejected(): void   // ts = time() - 600
public function test_missing_field_is_rejected(): void     // unset receipt
public function test_return_handler_does_not_fulfil_without_polling(): void
```

The last one matters most: fake `GET /orders/{id}` returning `attempted` while
the redirect says `paid`, and assert nothing was fulfilled.
