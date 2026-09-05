---
name: citi-virtual-card-lifecycle
description: Create, modify and cancel a Citi Virtual Card Account, and subscribe to authorisation events rather than polling for them.
api: Citi Commercial Cards and Virtual Card Accounts APIs
operations:
  - create
  - modify
  - cancel
  - createSubscription
  - getSubscription
  - updateSubscription
  - deleteSubscription
  - createDispute
  - getDisputeStatus
generated: '2026-09-05'
method: generated
source: openapi/citi-virtual-cards-lifecycle-v4-openapi.yaml, openapi/citi-vcaeventssubscriptions-openapi.yaml, openapi/citi-card-disputes-openapi.yaml
---

# Run a Virtual Card Account end to end

Base: `https://tts.apib2b.citi.com/tts/cards/vca/v2` (lifecycle) and
`https://tts.apib2b.citi.com/tts/cards` (subscriptions).

## Steps

1. **Authenticate** — see `citi-authenticate`.

2. **Create the VCA.** `create` (`POST /create`) places a VCA creation request for secure purchasing.

3. **Amend it.** `modify` (`POST /modify`) places a VCA update request.

4. **Retire it.** `cancel` (`POST /cancel`). Note what this actually does: it **turns the VCA
   inactive so it stops accepting payment requests**. It is a suspension of future authorisations,
   not an unwind of authorisations that already happened. Money already spent on the card is not
   recovered by cancelling it.

5. **Subscribe to events** rather than polling:
   - `createSubscription` (`POST /vca/v2/events/subscriptions`)
   - `getSubscription`, `updateSubscription`
   - `deleteSubscription` (`DELETE /vca/v2/events/subscriptions/{subscriptionId}`)

   If you cannot host a webhook receiver, `openapi/citi-vcagetnotifications-openapi.yaml` exposes a
   pull equivalent.

6. **Dispute a settled transaction.** `createDispute` (`POST /v1/cases`) on
   `https://tts.apib2b.citi.com/tts/cards/disputes`, then `getDisputeStatus` (`GET /v1/status`)
   using the Acquirer Reference Number or Case ID. Dispute windows are set by the card scheme and
   are **not** published in this contract.

## Rules that bite

- The VCA family publishes two concurrent major versions — v1 and 2.0.0 — plus separate contracts for
  payment intermediaries, mobile virtual cards and distributed-partner onboarding. Read
  `apis.yml` and pick the contract that matches your programme; the operation names repeat across them.
- `cancel` is reversible only in the sense that you can create a new VCA. There is no un-cancel.
- The APAC disputes host differs (`/tts/apac/cards/disputes`).
