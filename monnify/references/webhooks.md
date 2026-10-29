# Webhooks

Monnify POSTs JSON to URLs you set in the dashboard (**Developer → Webhook URLs**; separate URLs for transaction completion, refund completion, disbursement and settlement).

## Payload shape

```json
{
  "eventType": "SUCCESSFUL_TRANSACTION",
  "eventData": {
    "transactionReference": "MNFY|...",
    "paymentReference": "order_8812_1730060000",
    "amountPaid": 5000,
    "totalPayable": 5000,
    "settlementAmount": 4925,
    "paidOn": "2026-10-27 14:22:10.0",
    "paymentStatus": "PAID",
    "paymentMethod": "ACCOUNT_TRANSFER",
    "currency": "NGN",
    "product": { "type": "WEB_SDK", "reference": "order_8812_1730060000" },
    "customer": { "name": "Ada Obi", "email": "ada@example.com" },
    "metaData": { "orderId": "8812" }
  }
}
```

`product.type` tells you the source, e.g. `WEB_SDK`, `API_NOTIFICATION`, `RESERVED_ACCOUNT`, `INVOICE`. For reserved accounts, `product.reference` is the `accountReference`.

## Event types

| eventType | When |
|---|---|
| `SUCCESSFUL_TRANSACTION` | A collection succeeded (checkout, reserved account, invoice, offline payment) |
| `SUCCESSFUL_DISBURSEMENT` | A transfer completed |
| `FAILED_DISBURSEMENT` | A transfer failed |
| `REVERSED_DISBURSEMENT` | A transfer was reversed back to your wallet |
| `SUCCESSFUL_REFUND` | A refund was credited to the customer |
| `FAILED_REFUND` | A refund could not be processed |
| Settlement completion, mandate status change, wallet activity, rejected payment | See the Event Types page for exact names |

Exact names for that last row: check `https://developers.monnify.com/docs/webhooks/event-types`. Switch on `eventType` and log-and-ignore anything unknown rather than failing.

## Verifying the signature (required)

Header: `monnify-signature`. Value: lowercase hex **HMAC-SHA512** of the **raw request body**, keyed with your **Secret Key** (the same key used for login; there is no separate webhook secret).

You must hash the exact bytes received. Re-serializing parsed JSON changes spacing/ordering and the hash will not match.

### Node.js / Express

```js
import crypto from "node:crypto";
import express from "express";

const app = express();

app.post(
  "/webhooks/monnify",
  express.raw({ type: "application/json" }),   // keep the raw body
  async (req, res) => {
    const signature = req.get("monnify-signature") ?? "";
    const expected = crypto
      .createHmac("sha512", process.env.MONNIFY_SECRET_KEY)
      .update(req.body)                          // Buffer of raw bytes
      .digest("hex");

    const valid =
      signature.length === expected.length &&
      crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
    if (!valid) return res.sendStatus(401);

    res.sendStatus(200);                          // acknowledge fast

    const event = JSON.parse(req.body.toString("utf8"));
    await queue.add("monnify-event", event);      // process asynchronously, idempotently
  }
);
```

### Next.js route handler

```ts
export async function POST(req: Request) {
  const raw = await req.text();
  const expected = crypto.createHmac("sha512", process.env.MONNIFY_SECRET_KEY!).update(raw).digest("hex");
  if (req.headers.get("monnify-signature") !== expected) return new Response(null, { status: 401 });
  const event = JSON.parse(raw);
  // enqueue / process idempotently
  return new Response(null, { status: 200 });
}
```

### Python / Flask

```python
import hmac, hashlib, os
from flask import Flask, request, abort

app = Flask(__name__)

@app.post("/webhooks/monnify")
def monnify_webhook():
    raw = request.get_data()
    expected = hmac.new(os.environ["MONNIFY_SECRET_KEY"].encode(), raw, hashlib.sha512).hexdigest()
    if not hmac.compare_digest(expected, request.headers.get("monnify-signature", "")):
        abort(401)
    event = request.get_json()
    # enqueue / process idempotently
    return "", 200
```

### PHP / Laravel

```php
$expected = hash_hmac('sha512', $request->getContent(), config('services.monnify.secret'));
abort_unless(hash_equals($expected, $request->header('monnify-signature', '')), 401);
```

## Best practices

1. **Verify the signature** before trusting anything.
2. **Return 200 quickly**, then do the work. Monnify retries when it does not get a 200 or the request times out, so slow handlers cause duplicates.
3. **De-duplicate**: store `eventData.transactionReference` (or transfer/refund reference) with a unique constraint and skip repeats.
4. **Optionally allow-list** Monnify's webhook IP `35.242.133.146` (confirm against the docs; IPs can change, so do not rely on this alone).
5. **Confirm high-value events** with a status call before giving value.
6. Local development: expose your server with a tunnel (ngrok, cloudflared) and set that URL in the sandbox dashboard.
