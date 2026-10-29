# Errors and common mistakes

## Response envelope

Success: `requestSuccessful: true`, `responseCode: "0"`. Failure usually has `requestSuccessful: false`, a non-zero `responseCode` (often `"99"`) and a human-readable `responseMessage`. Always log `responseMessage`; it is usually the most specific clue.

| HTTP | Typical meaning |
|---|---|
| 400 | Validation error (missing field, bad format, duplicate reference) |
| 401 | Bad/expired token, wrong environment keys, missing header |
| 404 | Reference not found (check URL-encoding of `|`) |
| 422 | Business rule rejected (e.g. invalid contract code, feature not enabled) |
| 500 | Monnify-side; retry with backoff, then contact support |

Refund-specific codes seen in the docs include `R11` (name inquiry network error, retry later) and `M01`/`M02` (system error, contact support).

## Mistakes that come up again and again

1. **Sandbox keys against the live URL** (or the reverse). Symptom: 401 on login.
2. **Fulfilling on the redirect or `onComplete`.** Anyone can open your success URL. Verify server-side.
3. **Not checking the amount.** A `PAID` status for ₦100 on a ₦10,000 order is still `PAID`. Compare `amountPaid` to the expected amount.
4. **Reused references.** A new attempt needs a new `paymentReference`. Checkout fails when a reference was already used.
5. **Unencoded transaction references.** `MNFY|...` must be `MNFY%7C...` in URLs.
6. **Hashing parsed JSON for the webhook signature.** Hash the raw body bytes.
7. **Slow webhook handlers.** Monnify retries; process asynchronously and de-duplicate.
8. **Secret Key in frontend code or a mobile app bundle.** Rotate it immediately if this happens.
9. **Treating `PENDING` transfers as failed** and retrying with a new reference → double payout. Poll the original reference.
10. **Expecting refunds to virtual accounts.** Not supported; collect a real destination account.
11. **Hard-coding partner bank lists** for reserved accounts. Read what Monnify returns.
12. **Forgetting to switch every credential when going live**: base URL, API Key, Secret Key, Contract Code, wallet number, and webhook URLs in the live dashboard.
13. **Late bank-transfer payments.** Customers sometimes pay after the checkout account expires. Monnify still notifies you; decide whether to fulfil, refund or credit, rather than ignoring the webhook.

## Getting help

- Docs: `https://developers.monnify.com`
- API reference (OpenAPI): `https://developers.monnify.com/collection/monnify-collection.yml`
- Integration support: `integration-support@monnify.com`
- Feature access (transfers, wallets, sub-accounts): `sales@monnify.com`
- Monnify developer community on Slack (link on the docs site)
