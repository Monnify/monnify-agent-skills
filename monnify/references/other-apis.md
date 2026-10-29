# Other APIs

All endpoints use `Authorization: Bearer <token>` and the standard response envelope. Several need activation in live mode; noted per section.

## Refunds (live use needs approval)

```
POST /api/v1/refunds/initiate-refund
{
  "transactionReference": "MNFY|...",
  "refundReference": "refund_8812_1",
  "refundAmount": 5000,
  "refundReason": "Out of stock",
  "customerNote": "Refund for order #8812"
}
```

- Full or partial (`refundAmount` less than the amount paid).
- Monnify refunds to the account that paid. **It cannot refund to a virtual account**, so for payments that came from reserved/virtual accounts, collect a real bank account and add `destinationAccountNumber` and `destinationAccountBankCode`.
- Status: `GET /api/v1/refunds/{refundReference}`. List: `GET /api/v1/refunds?page=0&size=20`.
- Final state arrives as `SUCCESSFUL_REFUND` / `FAILED_REFUND` webhooks.

## Sub-accounts and split payments (live use needs approval)

Create the sub-accounts once (body is an **array**):

```
POST /api/v1/sub-accounts
[{ "currencyCode": "NGN", "bankCode": "058", "accountNumber": "0123456789",
   "email": "partner@example.com", "defaultSplitPercentage": 20 }]
```

Response gives each a `subAccountCode` (e.g. `MFY_SUB_...`). Then split a transaction or reserved account with:

```json
"incomeSplitConfig": [
  { "subAccountCode": "MFY_SUB_319452883228", "feePercentage": 10.5,
    "splitPercentage": 20, "feeBearer": true }
]
```

Use either `splitPercentage` or `splitAmount`. `feeBearer` decides who absorbs Monnify's fee. Manage with `GET`/`PUT /api/v1/sub-accounts` and `DELETE /api/v1/sub-accounts/{subAccountCode}`.

## Invoices

```
POST /api/v1/invoice/create
{ "amount": 25000, "invoiceReference": "inv_2026_044", "description": "Consulting, Oct",
  "currencyCode": "NGN", "contractCode": "1234567890", "customerEmail": "client@example.com",
  "customerName": "Acme Ltd", "expiryDate": "2026-11-30 23:59:59", "redirectUrl": "https://..." }
```

Including an `accountReference` (of an INVOICE reserved account) makes it a **static** invoice; leaving it out makes a **dynamic** one. Details: `GET /api/v1/invoice/{ref}/details`. List: `GET /api/v1/invoice/all`. Cancel: `DELETE /api/v1/invoice/{ref}/cancel`.

## Recurring card payments (card tokens)

After a successful card payment, the transaction carries a `cardToken`. Store it **with the customer email used in that first charge**. Charge later:

```
POST /api/v1/merchant/cards/charge-card-token
{ "cardToken": "MNFY_...", "amount": 3000, "customerName": "Ada Obi",
  "customerEmail": "ada@example.com", "paymentReference": "sub_ada_2026_11",
  "paymentDescription": "Monthly plan", "currencyCode": "NGN", "contractCode": "1234567890" }
```

## Direct debit mandates

| Purpose | Endpoint |
|---|---|
| Create mandate | `POST /api/v1/direct-debit/mandate/create` |
| Mandate status | `GET /api/v1/direct-debit/mandate/?mandateReferences=<ref>` |
| Debit a mandate | `POST /api/v1/direct-debit/mandate/debit` |
| Debit status | `GET /api/v1/direct-debit/mandate/debit-status?paymentReference=<ref>` |
| Cancel | `PATCH /api/v1/direct-debit/mandate/cancel-mandate/{mandateCode}` |
| List | `GET /api/v1/direct-debit/mandates?startDate=...&endDate=...` (range at most 90 days; statuses `PENDING`, `ACTIVE`, `FAILED`, `CANCELLED`, `EXPIRED`) |

A mandate must be `ACTIVE` before you debit it.

## Limit profiles

Cap how much a reserved account can receive. Create with `POST /api/v1/limit-profile/` (single/daily transaction limits), list with `GET /api/v1/limit-profile/`, update with `PUT /api/v1/limit-profile/{limitProfileCode}`. Attach via `POST /api/v1/bank-transfer/reserved-accounts/limit` (create) or `PUT` on the same path (update).

## Settlements

- Transactions in a settlement: `GET /api/v1/transactions/find-by-settlement-reference?reference=<settlementRef>&page=0&size=20`
- Settlement for one transaction: `GET /api/v1/settlement-detail?transactionReference=<url-encoded ref>`

## Verification APIs (live mode only)

| Purpose | Endpoint |
|---|---|
| Validate bank account (name enquiry) | `GET /api/v2/disbursements/account/validate?accountNumber=...&bankCode=...` |
| BVN details match | `POST /api/v1/vas/bvn-details-match` `{ "bvn", "name", "dateOfBirth", "mobileNo" }` |
| BVN ↔ account match | `POST /api/v1/vas/bvn-account-match` `{ "bankCode", "accountNumber", "bvn" }` |
| NIN lookup | `POST /api/v1/vas/nin-details` `{ "nin" }` |

These return sample data in sandbox; build the flow there, test real lookups in live.

## Bills payment (activate via integration-support@monnify.com)

Flow: categories → billers → products → validate customer → vend → requery.

| Step | Endpoint |
|---|---|
| Categories | `GET /api/v1/vas/bills-payment/biller-categories` |
| Billers | `GET /api/v1/vas/bills-payment/billers?category_code=CABLE_TV` |
| Products | `GET /api/v1/vas/bills-payment/biller-products?biller_code=...` |
| Validate customer | `POST /api/v1/vas/bills-payment/validate-customer` `{ "productCode", "customerId" }` |
| Vend | `POST /api/v1/vas/bills-payment/vend` `{ "productCode", "customerId", "amount", "emailAddress", "validationReference"? }` |
| Status | `GET /api/v1/vas/bills-payment/requery?reference=...` |

After validation, check `vendInstruction.requireValidationRef`. If `true`, pass the returned `validationReference` in the vend call; if `false`, omit it. `priceType` `FIXED` means use the product `price`; `OPEN` lets you set the amount.

## Wallets (needs approval)

| Purpose | Endpoint |
|---|---|
| Create customer wallet | `POST /api/v1/disbursements/wallet` |
| List wallets | `GET /api/v1/disbursements/wallet?walletReference=...` |
| Balance | `GET /api/v1/disbursements/wallet/balance?accountNumber=...` |
| Transactions | `GET /api/v1/disbursements/wallet/transactions?accountNumber=...` |

## Paycodes

Create a code a beneficiary can use to withdraw cash: `POST /api/v1/paycode` with `{ "beneficiaryName", "amount", "paycodeReference", "expiryDate", "clientId" }`. See the API reference for fetch/cancel endpoints.

## Reference data

- All banks (collections and transfers): `GET /api/v1/banks`
- Banks with USSD codes: `GET /api/v1/sdk/transactions/banks`

Cache bank lists; they change rarely.
