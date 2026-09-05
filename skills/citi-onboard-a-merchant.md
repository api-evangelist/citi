---
name: citi-onboard-a-merchant
description: Onboard a merchant onto Citi Gateway Services — create the merchant, upload verification documents, certify, activate a wallet, link an account and start paying out.
api: Citi Gateway Services API
operations:
  - merchantCreation
  - getMerchantId
  - fileUpload
  - submitCertification
  - getCertification
  - activateWallet
  - getWalletBalances
  - linkAccount
  - fxQuery
  - createPayout
  - getPayment
  - saveRfiResponse
  - submitRfiResponse
  - getTransactions
generated: '2026-09-05'
method: generated
source: openapi/citi-marketplace-management-openapi.yaml
---

# Onboard a merchant and pay them out

Gateway Services is the single densest contract in the Citi estate — 19 operations in one
specification — and it is the one that reads most like a normal platform API. Base:
`https://b2b.api.icg.citi.com/citiconnect/prod/gatewayservices` (sandbox:
`https://sandbox.b2b.api.icg.citi.com/citiconnect/sb/gatewayservices`).

## Steps

1. **Authenticate** — see `citi-authenticate`.

2. **Create the merchant.** `merchantCreation` (`POST /merchants/v1/onboard`). Retrieve the assigned
   identifier with `getMerchantId` (`GET /merchants/v1/onboard`).

3. **Upload KYC documents.** `fileUpload` (`POST /merchants/v1/documents/upload`). Retrieve them
   later with `downloadFile`.

4. **Expect an RFI loop.** Citi will come back with Requests For Information. Poll `queryRfi`
   (`GET /merchants/v1/rfi`), answer with `saveRfiResponse` (`POST /merchants/v1/rfi`) and commit
   with `submitRfiResponse` (`POST /merchants/v1/rfi/submit`). **Budget for this** — it is a
   compliance conversation modelled as an API, and it is where onboarding stalls.

5. **Certify.** `submitCertification` (`POST /merchants/v1/certification`), check with
   `getCertification`.

6. **Activate a wallet.** `activateWallet` (`POST /merchants/v1/wallets/activation`) creates the
   virtual-account wallet. Check state with `getWallet` and funds with `getWalletBalances`.

7. **Link a payout account.** `linkAccount` (`POST /merchants/v1/link-accounts`), verify with
   `getLinkedAccount`.

8. **Pay out.** Optionally price the leg first with `fxQuery` (`POST /merchants/v1/fx-rate`), then
   `createPayout` (`POST /merchants/v1/payments`). Track with `getPayment`.

9. **Reconcile.** `getTransactions` (`GET /merchants/v1/transactions`) and `getBalanceReport`
   (`GET /merchants/v1/balance-report`).

## Rules that bite

- `createPayout` does **not** declare an `Idempotency-Id` header. Treat a timeout as unknown, and
  reconcile with `getPayment` and `getTransactions` before resubmitting.
- There is no documented reversal for `createPayout` in this contract. If you need refundability,
  the express-payments surface (`expressRefundInitiation`) is the one that publishes it.
- Wallet activation is gated on certification, which is gated on document verification. The order
  above is not optional.
