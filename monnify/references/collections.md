# Collections: one-time payments

Every one-time payment follows the same lifecycle, whatever the method:

1. **Initialize** a transaction (SDK or API) → Monnify returns a `transactionReference` (and a `checkoutUrl`).
2. The customer **pays** by card, bank transfer, USSD or Pay with Bank.
3. Monnify sends a `SUCCESSFUL_TRANSACTION` **webhook** and/or redirects the customer to `redirectUrl`.
4. Your server **verifies** the status and amount, then gives value.

Two integration paths:

| Path | When | Server auth needed |
|---|---|---|
| **Checkout SDK** (modal in the page) | Most web apps, frontend-first setups | No (to open checkout). Yes, to verify. |
| **API-first** (init on server, redirect to `checkoutUrl` or charge directly) | Full server control, mobile backends, custom UIs | Yes |

## Path A: Checkout SDK

Load the script and call `MonnifySDK.initialize`. Only the API Key and Contract Code go to the browser.

```html
<script src="https://sdk.monnify.com/plugin/monnify.js"></script>
<script>
  function payWithMonnify() {
    MonnifySDK.initialize({
      amount: 5000,
      currency: "NGN",
      reference: "order_8812_" + Date.now(),   // better: get this from your server
      customerFullName: "Ada Obi",
      customerEmail: "ada@example.com",
      apiKey: "MK_TEST_XXXXXXXXXX",
      contractCode: "1234567890",
      paymentDescription: "Order #8812",
      // paymentMethods: ["CARD", "ACCOUNT_TRANSFER", "USSD", "PAY_WITH_BANK"],
      onLoadStart: () => {},
      onLoadComplete: () => {},
      onComplete: (response) => {
        // NOT proof of payment. Send response.paymentReference / transactionReference
        // to your server and verify there.
        fetch("/api/payments/verify", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ paymentReference: response.paymentReference,
                                 transactionReference: response.transactionReference }),
        });
      },
      onClose: (data) => {},
    });
  }
</script>
```

Framework packages (React, Vue, Angular) exist on npm and wrap the same flow with a class (`new Monnify(apiKey, contractCode)` → `initializePayment(...)`). Check the Monnify docs for the current package name before installing.

## Path B: API-first

### Initialize Transaction

```
POST /api/v1/merchant/transactions/init-transaction
Authorization: Bearer <token>
```

```json
{
  "amount": 5000,
  "customerName": "Ada Obi",
  "customerEmail": "ada@example.com",
  "paymentReference": "order_8812_1730060000",
  "paymentDescription": "Order #8812",
  "currencyCode": "NGN",
  "contractCode": "1234567890",
  "redirectUrl": "https://shop.example.com/payments/return",
  "paymentMethods": ["CARD", "ACCOUNT_TRANSFER"],
  "metaData": { "orderId": "8812" }
}
```

- `amount`, `customerEmail`, `paymentReference`, `currencyCode`, `contractCode` are the core fields. `customerName`, `paymentDescription`, `redirectUrl` are strongly recommended.
- `paymentMethods` limits what the checkout shows. Omit it to show everything enabled on the contract. Values include `CARD`, `ACCOUNT_TRANSFER`, `USSD`, `PHONE_NUMBER`, `PAY_WITH_BANK`.
- `incomeSplitConfig` (array) splits this payment across sub-accounts; see `other-apis.md`.
- `metaData` is echoed back in webhooks; use it to carry your own IDs.

Response body includes `transactionReference`, `paymentReference`, `merchantName`, `apiKey`, `enabledPaymentMethod` and **`checkoutUrl`**. Redirect the customer to `checkoutUrl`.

**Security note from the API reference:** confirm the values in the response (especially `amount`) match what you sent. A tampered request can change them.

Checkout URLs expire (about 40 minutes). Initialize a fresh transaction per attempt.

### Pay by bank transfer without the hosted page (dynamic account)

```
POST /api/v1/merchant/bank-transfer/init-payment
{ "transactionReference": "MNFY|...", "bankCode": "058" }
```

Returns a one-time account number, bank name, amount and expiry for the customer to transfer to. `bankCode` is optional and is used to produce a USSD string for that bank.

### USSD

```
POST /api/v1/merchant/ussd/initialize
{ "transactionReference": "MNFY|...", "bankUssdCode": "737" }
```

Use `GET /api/v1/sdk/transactions/banks` to list banks with their USSD short codes.

### Pay with Bank

Add `PAY_WITH_BANK` to `paymentMethods` on the hosted checkout and Monnify handles the flow. For a custom UI: initialize the transaction, call the Pay with Bank init endpoint with the `transactionReference`, then redirect the customer to the returned `authorizationUrl`, where they enter their account number and an OTP. Returning from that page does **not** mean the payment succeeded; verify.

### Direct card charge (PCI-DSS required)

Only for merchants with PCI-DSS certification and the feature enabled by Monnify support. Card data must never touch a non-compliant server.

```
POST /api/v1/merchant/cards/charge
{ "transactionReference": "MNFY|...", "collectionChannel": "API_NOTIFICATION",
  "card": { "number": "...", "expiryMonth": "12", "expiryYear": "2025", "pin": "1234", "cvv": "123" } }
```

`responseBody.status` tells you the next step:

| status | Next step |
|---|---|
| `SUCCESS` | Verify, then fulfil |
| `OTP_AUTHORIZATION_REQUIRED` | `POST /api/v1/merchant/cards/otp/authorize` with the `otpData.id`, the `transactionReference` and the customer's OTP |
| `BANK_AUTHORIZATION_REQUIRED` | 3DS: send the customer to `secure3dData.redirectUrl`, then `POST /api/v1/sdk/cards/secure-3d/authorize` |

## Verifying a payment (always do this)

By Monnify reference:

```
GET /api/v2/transactions/{urlEncodedTransactionReference}
```

By either reference:

```
GET /api/v2/merchant/transactions/query?paymentReference=order_8812_1730060000
GET /api/v2/merchant/transactions/query?transactionReference=MNFY%7C67%7C20220725111957%7C000283
```

Give value only when:

- `paymentStatus === "PAID"`, and
- `amountPaid` (or `amount`) equals the amount you expected for that order, and
- you have not already fulfilled this `transactionReference`.

Other `paymentStatus` values you will see: `PENDING`, `OVERPAID`, `PARTIALLY_PAID`, `EXPIRED`, `FAILED`, `CANCELLED`, `REVERSED`. Decide explicitly what your app does for `OVERPAID` and `PARTIALLY_PAID` (common with bank transfers, where the customer types the amount).

```js
export async function verifyPayment(paymentReference, expectedAmount) {
  const tx = await monnify(
    `/api/v2/merchant/transactions/query?paymentReference=${encodeURIComponent(paymentReference)}`
  );
  const ok = tx.paymentStatus === "PAID" && Number(tx.amountPaid) === Number(expectedAmount);
  return { ok, status: tx.paymentStatus, transactionReference: tx.transactionReference, tx };
}
```

## Listing transactions

```
GET /api/v1/transactions/search?page=0&size=20&paymentStatus=PAID&from=1717200000000&to=1719791999000
```

`from`/`to` are Unix timestamps in milliseconds. Pages start at 0. Results are in `responseBody.content`.
