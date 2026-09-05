---
name: citi-account-reporting
description: Pull Citi account details, real-time balances and statements, and subscribe to credit/debit notifications instead of polling.
api: Citi Account Reporting APIs
operations:
  - getAccountsByClientId
  - getBalancesByClientId
  - getAccountBalance
  - initiateAccountStatement
  - getAccountStatement
  - AccountNotificationSubscriptionRequest
generated: '2026-09-05'
method: generated
source: openapi/citi-accountsv5-openapi.yaml, openapi/citi-balances-api-openapi.yaml, openapi/citi-statements-api-openapi.yaml, openapi/citi-account-notifications-api-openapi.yaml
---

# Read balances, statements and account data

## Steps

1. **Authenticate** — see `citi-authenticate`.

2. **List the accounts you are entitled to.** `getAccountsByClientId` (`GET /accounts`) on
   `https://tts.apib2b.citi.com/citiconnect/prod/accountsservices/v5`. Entitlement is granted per
   client during onboarding; this call returns what your credentials can actually see, which is not
   necessarily every account your organisation holds.

3. **Get balances.** Two routes, and they are not the same:
   - `getBalancesByClientId` (`GET /balances`, Accounts v5) — balances across the entitled set.
   - `getAccountBalance` (`POST /v3/balance/inquiry` on `.../accountstatementservices`) — the
     dedicated Balances API. Responses can be delivered as JSON **or** XML.

4. **Statements are a two-step, asynchronous flow.** This is the part people get wrong:
   - `initiateAccountStatement` (`POST /statement/initiation`) asks Citi to produce the statement.
   - `getAccountStatement` (`POST /statement/retrieval`) fetches it once ready.

   Do not call retrieval immediately and treat an empty result as "no transactions".

5. **Stop polling — subscribe.** `AccountNotificationSubscriptionRequest`
   (`POST /v1/crdr/subscription`) registers your endpoint for credit/debit notifications.
   See `asyncapi/citi-webhooks.yml` for the event surface.

6. **Block or filter an account** with `Blocks` (`POST /blocksandfilters`), inspect with
   `getBlockAndFiltersStatus`, and reverse with `deleteBlockAndFilter`.

## Rules that bite

- Statement content is ISO 20022 `camt.052.001.02` style. Parse the message, not a bespoke shape.
- `Region` and `Country` headers select the servicing entity. The same account number in a different
  region is a different account.
- Accounts is published as both **V1.4.2** (`/accountsservices/v4`) and **V5** (`/accountsservices/v5`)
  concurrently. Pin the version in your base URL; they are separate contracts, not a rolling upgrade.
- Statements v2 publishes a dedicated mock server at
  `https://tts.sandbox.apib2b.citi.com/citiconnect/sb/accountstatementservices/v1/mock`.
