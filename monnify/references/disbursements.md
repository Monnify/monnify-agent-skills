# Disbursements (transfers / payouts)

Transfers move money from your Monnify wallet to any Nigerian bank account. **The feature must be enabled for your merchant** (regulatory requirement). Request access via `sales@monnify.com`. Without it, calls fail even with valid credentials.

You need your **wallet account number** (`sourceAccountNumber`), from the dashboard.

## Before paying: validate the destination

```
GET /api/v2/disbursements/account/validate?accountNumber=0123456789&bankCode=058
```

Returns the account name. Show it to the user and compare it to what they expect before sending. Bank codes come from `GET /api/v1/banks`.

## Single transfer

```
POST /api/v2/disbursements/single
```

```json
{
  "amount": 20000,
  "reference": "payout_5531_1730060000",
  "narration": "Vendor payout #5531",
  "destinationBankCode": "058",
  "destinationAccountNumber": "0123456789",
  "currency": "NGN",
  "sourceAccountNumber": "3934178936"
}
```

`reference` must be unique. If a request times out, **do not resend with a new reference**; query the status of the same reference first, or you may pay twice.

### OTP (two-factor) authorization

By default, merchants have transfer OTP turned on. The response then has `status: "PENDING_AUTHORIZATION"` and an OTP is emailed to the account owner. Complete it with:

```
POST /api/v2/disbursements/single/validate-otp
{ "reference": "payout_5531_1730060000", "authorizationCode": "491763" }
```

Resend: `POST /api/v2/disbursements/single/resend-otp` with `{ "reference": "..." }`.

Merchants who disable OTP in the dashboard get the final state directly. Fully automated payouts usually need OTP disabled; that is a business decision for the account owner.

### Status

```
GET /api/v2/disbursements/single/summary?reference=payout_5531_1730060000
```

States: `PENDING`, `SUCCESS`, `FAILED`, `REVERSED`, `PENDING_AUTHORIZATION`, `OTP_EMAIL_DISPATCH_FAILED`.

`PENDING` is normal; final state arrives by webhook (`SUCCESSFUL_DISBURSEMENT`, `FAILED_DISBURSEMENT`, `REVERSED_DISBURSEMENT`). A `REVERSED` transfer means money came back to your wallet after first appearing successful; your ledger must handle that.

## Bulk transfer

```
POST /api/v2/disbursements/batch
```

```json
{
  "title": "October salaries",
  "batchReference": "batch_2026_10",
  "narration": "Salary Oct 2026",
  "sourceAccountNumber": "3934178936",
  "onValidationFailure": "CONTINUE",
  "notificationInterval": 25,
  "transactionList": [
    { "amount": 150000, "reference": "sal_2026_10_001", "narration": "Salary - Ada",
      "destinationBankCode": "058", "destinationAccountNumber": "0123456789", "currency": "NGN" }
  ]
}
```

- `onValidationFailure`: `CONTINUE` (skip bad rows) or `BREAK` (reject the batch).
- Authorize with `POST /api/v2/disbursements/batch/validate-otp` `{ "reference": "<batchReference>", "authorizationCode": "..." }`.
- Summary: `GET /api/v2/disbursements/batch/summary?reference=<batchReference>`.
- Items: `GET /api/v2/disbursements/bulk/{batchReference}/transactions`.

## Other endpoints

| Purpose | Endpoint |
|---|---|
| Wallet balance | `GET /api/v2/disbursements/wallet-balance?accountNumber=<wallet>` |
| List single transfers | `GET /api/v2/disbursements/single/transactions?pageNo=0&pageSize=20` |
| List bulk transfers | `GET /api/v2/disbursements/bulk/transactions?pageNo=0&pageSize=20` |
| Search all disbursements | `GET /api/v2/disbursements/search-transactions?sourceAccountNumber=<wallet>&startDate=<ms>&endDate=<ms>` |

## Safe payout checklist

- Check wallet balance before large batches.
- Persist the transfer row (with `reference`, status `INITIATED`) **before** calling Monnify.
- Treat timeouts as "unknown", then poll the same reference.
- Update state only from webhooks or status checks, never from the initiate response alone.
