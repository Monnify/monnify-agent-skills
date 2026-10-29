# Reserved (dedicated virtual) accounts

A reserved account is a permanent NUBAN account number assigned to one of your customers. Anything they transfer into it lands in your Monnify account and triggers a `SUCCESSFUL_TRANSACTION` webhook tagged with that customer. Typical uses: wallet funding, school fees, rent, savings, B2B collections.

## Create (general)

```
POST /api/v2/bank-transfer/reserved-accounts
Authorization: Bearer <token>
```

```json
{
  "accountReference": "user_10231",
  "accountName": "Ada Obi",
  "currencyCode": "NGN",
  "contractCode": "1234567890",
  "customerEmail": "ada@example.com",
  "customerName": "Ada Obi",
  "bvn": "21212121212",
  "getAllAvailableBanks": true
}
```

- `accountReference` is your stable ID for this customer (e.g. your user ID). It must be unique; reusing one returns an error, so creation should be "get or create".
- **KYC:** provide the customer's `bvn` (11 digits) and/or `nin`. Accounts created without KYC may be restricted; link it later with the KYC endpoint below.
- `getAllAvailableBanks: true` creates accounts at every partner bank. Set it to `false` and pass `"preferredBanks": ["50515", "035"]` to choose. Known partner bank codes include `50515` (Moniepoint MFB), `035` (Wema), `232` (Sterling), `058` (GTBank); availability changes, so do not hard-code a list as the only option.
- `incomeSplitConfig` (optional) splits every inflow to sub-accounts.
- `restrictPaymentSource: true` + `allowedPaymentSources` limits who can pay in (by `bvns`, `bankAccounts`, or `accountNames`).

Response `responseBody.accounts` is an array of `{ bankCode, bankName, accountNumber, accountName }`. Show all of them to the customer. Also stored: `reservationReference`, `status` (`ACTIVE`), `reservedAccountType` (`GENERAL`).

Do not assume the returned `accountName` equals what you sent; banks may truncate or prefix it. Display what Monnify returns.

## Other operations

| Operation | Endpoint |
|---|---|
| Get details | `GET /api/v2/bank-transfer/reserved-accounts/{accountReference}` |
| Add accounts at more banks | `PUT /api/v1/bank-transfer/reserved-accounts/add-linked-accounts/{accountReference}` body `{ "getAllAvailableBanks": false, "preferredBanks": ["232"] }` |
| Link / update BVN or NIN | `PUT /api/v1/bank-transfer/reserved-accounts/{accountReference}/kyc-info` body `{ "bvn": "...", "nin": "..." }` |
| Update BVN (older) | `PUT /api/v1/bank-transfer/reserved-accounts/update-customer-bvn/{accountReference}` |
| Restrict payment sources | `PUT /api/v1/bank-transfer/reserved-accounts/update-payment-source-filter/{accountReference}` |
| Update split config | `PUT /api/v1/bank-transfer/reserved-accounts/update-income-split-config/{accountReference}` (body is an array) |
| List inflows | `GET /api/v1/bank-transfer/reserved-accounts/transactions?accountReference=...&page=0&size=10` |
| Deallocate (delete) | `DELETE /api/v1/bank-transfer/reserved-accounts/reference/{accountReference}` |
| With a transaction limit | `POST /api/v1/bank-transfer/reserved-accounts/limit` (needs a Limit Profile, see `other-apis.md`) |

## Invoice reserved accounts

`POST /api/v1/bank-transfer/reserved-accounts` (v1) with `"reservedAccountType": "INVOICE"` creates an account that only accepts payment once attached to an invoice. Pass its `accountReference` when creating a static invoice.

## Crediting the customer correctly

When the webhook arrives:

1. Verify the signature (`webhooks.md`).
2. Read `eventData.product.reference` (your `accountReference`) and `eventData.product.type == "RESERVED_ACCOUNT"` to find the customer.
3. Credit `eventData.amountPaid` (or `settlementAmount` if you pass fees to the customer; decide once and document it).
4. Store `eventData.transactionReference` with a unique constraint so a resent webhook cannot double-credit.
5. Optionally confirm with `GET /api/v2/transactions/{ref}` before crediting large amounts.
