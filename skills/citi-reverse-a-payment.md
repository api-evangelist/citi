---
name: citi-reverse-a-payment
description: Take back money that has already been instructed — cancel an in-flight payment with an ISO 20022 camt.056, or refund one that has settled — and know which of the two you are actually able to do.
api: Citi Outgoing Payments APIs
operations:
  - FICancellationAPI
  - expressRefundInitiation
  - findRefundById
  - refundPayments
generated: '2026-09-05'
method: generated
source: openapi/citi-paymentcancellation-json-openapi.yaml, openapi/citi-express-payments-api-openapi.yaml, openapi/citi-payment-refund-openapi.yaml, openapi/citi-digitalpaymentscollectionsv12-openapi.yaml
---

# Reverse a Citi payment

There are two different actions here and they are not interchangeable.

- **Cancellation** stops a payment that has not settled. It is a *request*, not a guarantee.
- **Refund** sends the money back after settlement. It is a *new outgoing payment* with its own
  identifier, its own fees and its own failure modes.

**Citi does not publish a window for either one.** No contract and no published guide states how long
after initiation a cancellation will be honoured, or how long after settlement a refund is accepted.
Do not assume one, and do not promise a caller a deadline the bank has not stated.

## Cancel an in-flight payment

1. Call `FICancellationAPI` (`POST /payments/stops`) on
   `https://tts.apib2b.citi.com/citiconnect/prod/paymentservices/v3`.
2. The body is an ISO 20022 **`camt.056.001.09`** FI to FI Payment Cancellation Request. You are
   asking the clearing chain to stop the payment; the answer comes back as a status report.
3. Treat a `202` as "the cancellation request was accepted", not "the payment is cancelled".
   Re-inquire with `paymentStatusInquiry` to find out what actually happened.
4. Reason codes that mean you were too late: `FF12` (original transaction not eligible for the
   requested return), `FF13` (request for cancellation not found).

## Refund a settled payment

- **Instant / express payments:** `expressRefundInitiation`
  (`POST /paymentservices/v1/refunds`). This is one of the ten operations in the estate that
  **does** accept an `Idempotency-Id` header — use it. Track the result with
  `findRefundById` (`GET /paymentservices/v1/refunds/{refund_id}`) or `findRefunds`.
- **Payment acceptance / collections:** `refundPayments`
  (`POST /digitalpayments/v1/payment-acceptance/refunds`), then `getrefundpaymentstatus`, and
  `refundUpdate` (`PATCH .../refunds/{id}`) to amend one.
- **Generic payment refund:** `POST /txrefund` on the payment services v3 host.

## Adjacent reversals elsewhere in the estate

| Thing you did | How you take it back |
|---|---|
| Created a mandate | `recallMandate` (`PATCH /v1/mandates/recall`), `cancelMandate` |
| Issued a virtual card | `cancel` (`POST /cancel`) — sets the VCA inactive |
| Placed an FX order | `POST /fxgateway/sync/ordercancel/api/v1`, WorldLink `cancelFx` |
| Issued a trade instrument | `POST /trade/modifyorcancel` |
| Created a virtual account | `deleteVirtualAccount` |
| Blocked an account | `deleteBlockAndFilter` |
| A card transaction already settled | `createDispute` (`POST /v1/cases`) — a scheme dispute, not a Citi refund |

## Rules that bite

- Cancellation success depends on clearing-system cut-off, which is market-specific and **not**
  published in the contract. The self-service surface exposes a Payment Cut-Off Time Inquiry
  (`GET /citiconnect/prod/selfservices/v1/payment/cutoff`) — call it if timing matters.
- A refund is a fresh payment. It can itself be rejected, with its own reason code.
- Card disputes run on card-scheme timelines set by the network, not by Citi.
