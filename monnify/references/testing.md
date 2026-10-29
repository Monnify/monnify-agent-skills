# Testing in sandbox

Base URL: `https://sandbox.monnify.com`. Use your own test keys from the dashboard (Test Mode). The public API reference also publishes shared test credentials for quick experiments:

- API Key: `MK_TEST_GC3B8XG2XX`
- Secret Key: `A663NRZA544DDPEM7KDN7Z8HRV6YXD8S`
- Contract Code: `5867418298`

Shared credentials are fine for trying calls, but webhooks go to whatever URL that shared account has configured. Use your own sandbox account to test webhooks.

## Test cards

| Scenario | Number | Expiry | PIN | CVV |
|---|---|---|---|---|
| Success, no OTP | `4111111111111111` | 10/2025 | 1234 | 123 |
| Requires OTP (use `123456`) | `5060995994247093` | 12/2025 | 1234 | 123 |
| Requires 3DS | `4000000000000002` | 12/2025 | 1234 | 123 |
| Fails | `4111111111111110` | 10/2025 | 1234 | 123 |

These come from the published API reference. If an expiry is rejected as past, check the docs for updated values.

## Bank transfer simulator

To complete a sandbox bank-transfer payment (checkout or reserved account), use the web simulator: `https://websim.sdk.monnify.com/#/bankingapp`. Enter the account number Monnify generated and the amount; Monnify then fires the webhook as if a real transfer arrived. This is the easiest way to test reserved-account crediting end to end.

## Things sandbox does not do

- Verification APIs (BVN, NIN, BVN-account match) return sample responses only.
- Transfers, wallets, bills payment and sub-accounts may need to be enabled on your sandbox account too; if you get permission errors, ask integration support.

## Suggested test plan for a new integration

1. Auth: get a token; confirm a wrong key gives 401.
2. Checkout: pay with the no-OTP card → webhook arrives → status shows `PAID`.
3. OTP and 3DS cards: complete each flow.
4. Failed card: confirm your UI and order state handle failure.
5. Bank transfer: initialize, pay a **different** amount in the simulator → handle `PARTIALLY_PAID` / `OVERPAID`.
6. Webhook: send the same payload twice → confirm only one fulfilment. Send a bad signature → 401.
7. Reserved account: create, pay in via simulator, confirm the right user is credited once.
