---
name: citi-initiate-and-track-payment
description: Initiate an ISO 20022 payment through CitiConnect, then follow it to a final settlement status using the payment status and enhanced inquiry operations.
api: Citi Outgoing Payments APIs
operations:
  - paymentInitiationUsingPOST
  - paymentStatusInquiry
  - getPaymentStatus
generated: '2026-09-05'
method: generated
source: openapi/citi-paymentinitiation-pain103-openapi.yaml, openapi/citi-payment-status-openapi.yaml, openapi/citi-paymentenhancedinquiry-json-openapi.yaml, openapi/citi-proof-of-payment-openapi.yaml
---

# Initiate a payment and track it to final status

Citi payment initiation is **asynchronous**. A `202` means Citi accepted your instruction, not that
money moved. Settlement is confirmed by a separate inquiry or by a webhook.

## Steps

1. **Authenticate** — see `citi-authenticate`. The payment services surface sits at
   `https://tts.apib2b.citi.com/citiconnect/prod/paymentservices/v3` (sandbox:
   `https://tts.sandbox.apib2b.citi.com/citiconnect/sb/paymentservices/v3`).

2. **Choose the message type.** Citi publishes a separate contract per ISO 20022 message:

   | Use case | Message | Spec |
   |---|---|---|
   | Corporate credit transfer | `pain.001.001.03` | `openapi/citi-paymentinitiation-pain103-openapi.yaml` |
   | FI to FI customer credit transfer | `pacs.008.001.08` | `openapi/citi-paymentinitiation-pacs008-openapi.yaml` |
   | FI credit transfer | `pacs.009.001.08` | `openapi/citi-paymentinitiation-pacs009-openapi.yaml` |
   | Bulk | — | `openapi/citi-bulk-payments-openapi.yaml` |

   Each is published in a JSON and an XML encoding. Pick one and set `Content-Type` to match —
   a mismatch returns `415`, which is declared on 130 operations across the estate.

3. **Submit.** Call `paymentInitiationUsingPOST` (`POST /payment/initiation`). Set the `Region`
   and `Country` headers for the servicing entity, and a `Req-Sys-Id` for your own tracing.
   Capture the returned end-to-end identifier — on the instant-payments surface this is the ISO 20022
   **`uetr`**, a 36-character reference that is your handle for everything that follows.

4. **Poll for status.** Call `paymentStatusInquiry` (`POST /payment/enhancedinquiry`) or
   `POST /payment/inquiry`. The response is an ISO 20022 `pain.002.001.03` / `pacs.002.001.10`
   status report. Read `TxInfAndSts.StsRsnInf.Rsn.Cd` for the reason code and map it through
   `errors/citi-decline-codes.yml`.

5. **Interpret the status honestly.**
   - `ACCC` — settled on the creditor account. Done.
   - `ACWP` — accepted without posting. Not done.
   - `PDNG` — deemed approved, final status awaited. **Not done.** Keep polling.
   - `RJCT` — rejected. Read the reason code; do not retry blindly.

6. **Get evidence if you need it.** `getPaymentStatus`
   (`GET /v1/proofofpayment/{transaction_ref_no}` on `.../paymentinsights`) returns a proof-of-payment
   PDF and XML report for a specific transaction.

## Rules that bite

- **`paymentInitiationUsingPOST` does not declare an `Idempotency-Id` header.** Only 10 operations in
  the whole estate do, and payment initiation is not one of them. A `504 Gateway Timeout` on a payment
  submission is therefore ambiguous — the instruction may have been accepted. **Do not blind-retry.**
  Inquire on your end-to-end identifier first, and only resubmit if the inquiry finds nothing.
- `429` is declared on 82 operations and almost never carries `RateLimit-*` headers or `Retry-After`.
  Back off exponentially.
- A timeout reason code (`AB01`, `AB06`, `AB11`, `FF11`) means the clearing system, not Citi, timed
  out. The payment may still complete.
