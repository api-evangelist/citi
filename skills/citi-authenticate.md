---
name: citi-authenticate
description: Obtain a Citi OAuth 2.0 access token and use it on any CitiConnect API call, choosing the right authentication endpoint version for the API you are calling.
api: Citi API Authentication Services
operations:
  - oAuthV3UsingPOST
  - oAuthV4UsingPOST
generated: '2026-09-05'
method: generated
source: openapi/citi-authentication-api-3-openapi.yaml, openapi/citi-authentication-api-4-openapi.yaml, https://developer.citi.com/apidocs/authentication/authentication-only-guide
---

# Authenticate against a Citi API

Every Citi institutional API is protected by OAuth 2.0 over mutual TLS. There is no single
token endpoint: Citi runs **four authentication versions concurrently (V1–V4)** and each product
contract names the one it requires. Read the `securitySchemes` block of the API you are calling
before you pick an endpoint — using the wrong version returns 401 with no useful hint.

## Before you start

You need all of these, and none of them are self-service:

- A registered developer account on `https://partner.citi.com/user/register`.
- A **second** developer account from the same organisation. Citi requires one account to upload
  the public keys and a different colleague to complete the certificate process.
- A Client ID and Client Secret.
- Your uploaded public-key certificate, and Citi's downloaded certificate.

## Steps

1. **Find the version the target API wants.** Open the API's OpenAPI file under `openapi/` and read
   `components.securitySchemes`. The `tokenUrl` tells you which of the four endpoints to call —
   for example `/authenticationservices/v3/oauth/token` or `/authenticationservices/v4/oauth/token`.

2. **Build the token request.** Call `oAuthV3UsingPOST`
   (`POST /authenticationservices/v3/oauth/token`) or `oAuthV4UsingPOST`
   (`POST /authenticationservices/v4/oauth/token`) on the environment host:

   - sandbox: `https://tts.sandbox.apib2b.citi.com/citiconnect/sb`
   - production: `https://tts.apib2b.citi.com/citiconnect/prod`

   Send `grant_type=client_credentials`, with `Authorization` carrying your Basic-encoded
   Client ID and Secret and `Content-Type: application/json`.

3. **Sign and encrypt — unless you are on V4.** For V1, V2 and V3 the payload must be signed and
   encrypted with asymmetric PKI keys before it is sent, and the response must be decrypted and its
   signature verified. **V4 removes this requirement.** If the API you are calling supports V4,
   prefer it: it is the only version where the request body is plain JSON.

4. **Use the token.** Put it on every subsequent call as `Authorization: Bearer <access_token>`.
   All calls must be HTTPS; plain HTTP and unauthenticated calls fail.

5. **Handle expiry.** Citi issues **no refresh token**. The published guidance is: renew the access
   token only when you receive a `401 Unauthorized`, then retry the original call once. Do not
   refresh on a timer, and do not treat a 401 as a permanent failure.

## Rules that bite

- Token length is explicitly not guaranteed stable. Store it as a variable-length string; do not
  size a column to today's token.
- `Region` and `Country` headers are required on many operations and select the servicing Citi
  legal entity. A token is not automatically valid for every region your organisation banks in.
- Production access requires completing relationship-managed onboarding with a Citi Relationship
  Manager. A sandbox token will never work against a `prod` host.
