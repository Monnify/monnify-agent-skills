---
name: monnify
description: Build, debug and review integrations with Monnify, Moniepoint's payments API for Nigerian businesses. Use whenever code or questions involve Monnify, api.monnify.com, sandbox.monnify.com, MonnifySDK, MK_TEST_/MK_PROD_ keys, contract codes, reserved (virtual) accounts, Monnify checkout, transfers/disbursements, refunds, sub-accounts, split payments, direct debit mandates, bills payment, or verifying Monnify webhooks (monnify-signature).
---

# Monnify integration skill

Monnify is Moniepoint's payment gateway. It lets a business **collect** money (card, bank transfer, USSD, Pay with Bank, dedicated virtual accounts) and **send** money (single and bulk transfers), plus refunds, split settlements, mandates, invoices, bills payment, wallets and identity verification.

Use this file to orient, then open the reference file for the task at hand. Do not load every reference up front.

| Task | Read |
|---|---|
| Keys, base URLs, getting a token, sandbox vs live | `references/auth-and-environments.md` |
| Accepting a one-time payment (Checkout SDK, checkout URL, card, bank transfer, USSD, Pay with Bank) and verifying it | `references/collections.md` |
| Giving each customer a permanent account number to pay into | `references/reserved-accounts.md` |
| Paying out to bank accounts, single or bulk, OTP authorization, wallet balance | `references/disbursements.md` |
| Receiving and verifying webhooks | `references/webhooks.md` |
| Refunds, sub-accounts/split payments, invoices, direct debit, card tokens, limit profiles, settlements, verification (BVN/NIN/account), bills payment, wallets, paycodes, bank lists | `references/other-apis.md` |
| Test cards, sandbox credentials, the bank-transfer simulator | `references/testing.md` |
| Errors, response envelope, and the mistakes developers make most | `references/errors-and-gotchas.md` |

The full machine-readable API reference (OpenAPI 3.1) lives at `https://developers.monnify.com/collection/monnify-collection.yml`. Human docs live at `https://developers.monnify.com`. If a field name or path here disagrees with those, trust the live docs and say so.

## The five rules that prevent most broken integrations

1. **Never ship secrets to the browser.** The Secret Key only lives on the server. The frontend Checkout SDK needs only the API Key and Contract Code.
2. **Match keys to the base URL.** `MK_TEST_...` keys work only against `https://sandbox.monnify.com`; live keys only against `https://api.monnify.com`. Mismatches are the single most common cause of 401s.
3. **Never give value on the redirect or the SDK `onComplete` callback alone.** Confirm server-side: call Get Transaction Status and check `paymentStatus == "PAID"` *and* that `amountPaid` equals what you expected. Treat the webhook as the primary signal and the status call as the confirmation.
4. **Verify every webhook.** Compute HMAC-SHA512 of the *raw* request body using the Secret Key and compare it to the `monnify-signature` header. Respond `200` fast, then process. De-duplicate by `transactionReference`.
5. **References are single-use.** Every `paymentReference`, `accountReference`, transfer `reference` and `refundReference` must be unique per merchant. Generate them server-side (e.g. a UUID or `order_<id>_<timestamp>`); never reuse one for a retry of a different attempt.

## How every request works

```
POST {BASE_URL}/api/v1/auth/login
Authorization: Basic base64(API_KEY:SECRET_KEY)
→ responseBody.accessToken   (valid ~1 hour; cache it and refresh before expiry)

Then on every other call:
Authorization: Bearer <accessToken>
Content-Type: application/json
```

Every response uses one envelope:

```json
{ "requestSuccessful": true, "responseMessage": "success", "responseCode": "0", "responseBody": { } }
```

Check `requestSuccessful` and `responseCode == "0"`, not only the HTTP status.

Monnify transaction references contain `|` (e.g. `MNFY|67|20220725111957|000283`). **URL-encode them** (`MNFY%7C67%7C...`) when they go in a path or query string.

## Picking the right product

- *"Customer pays once for an order"* → Collections: Checkout SDK (frontend) or Initialize Transaction + `checkoutUrl` (server). See `collections.md`.
- *"Each user needs their own account number to fund a wallet"* → Reserved accounts. See `reserved-accounts.md`.
- *"Pay vendors, staff or users out"* → Disbursements (needs feature access from Monnify). See `disbursements.md`.
- *"Split each payment between us and a partner"* → Sub-accounts + `incomeSplitConfig`. See `other-apis.md`.
- *"Charge a customer every month"* → Card tokenization (`charge-card-token`) or Direct Debit mandates. See `other-apis.md`.
- *"Check a customer's identity or account name before paying them"* → Verification APIs / Validate Bank Account. See `other-apis.md`.

Some products (transfers, sub-accounts in live, refunds in live, wallets, bills payment, direct card charge) need activation from Monnify. If a call fails with a permission-style error in live mode, tell the developer to contact `integration-support@monnify.com` or their relationship manager rather than retrying.

## When writing code

- Read credentials from environment variables: `MONNIFY_API_KEY`, `MONNIFY_SECRET_KEY`, `MONNIFY_CONTRACT_CODE`, `MONNIFY_BASE_URL` (and `MONNIFY_WALLET_ACCOUNT_NUMBER` for transfers).
- Wrap auth in a small client that caches the token and refreshes on expiry or on a 401.
- Amounts are in **naira as a number** (e.g. `1500.50`), not kobo. Currency is `NGN`.
- Store Monnify's `transactionReference` alongside your own `paymentReference`; you need both for support, refunds and reconciliation.
- Make webhook and fulfilment handlers idempotent.
- Prefer the official SDKs when the stack has one (e.g. `monnify-python` on PyPI). Otherwise plain HTTP is fine; the API is small.

## Related tools

- **Monnify MCP server** (`@monnify/mcp-server`) lets an AI client act on a Monnify account directly (create reserved accounts, initiate payments, query transactions). Use it when the user wants to *operate* Monnify from chat. This skill is for *building* integrations. Setup: `https://developers.monnify.com/docs/integration/mcp-server`.
- Developer support: `integration-support@monnify.com`. Feature access: `sales@monnify.com`.
