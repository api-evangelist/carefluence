---
name: carefluence-smart-app-authorization
description: >-
  Obtain a SMART on FHIR access token from the Carefluence authorization server
  and confirm what the token is allowed to read, before calling any clinical
  resource.
api: Carefluence Open API R4
generated: '2026-09-02'
method: generated
source: >-
  Grounded in https://core.carefluence.com/cf.admin.core/.well-known/openid-configuration
  (HTTP 200), https://classic.carefluence.com/r4/.well-known/smart-configuration
  (HTTP 200) and the "Security & Authorization Flow" and "Client Registration"
  sections of the published Postman collection at https://api.carefluence.com/.
  Every URL below was fetched or is published verbatim by Carefluence.
operations:
  - 'GET https://classic.carefluence.com/r4/.well-known/smart-configuration'
  - 'GET https://core.carefluence.com/cf.admin.core/.well-known/openid-configuration'
  - 'GET https://core.carefluence.com/cf.admin.core/connect/authorize'
  - 'POST https://core.carefluence.com/cf.admin.core/connect/token'
  - 'POST https://core.carefluence.com/cf.admin.core/connect/introspect'
  - 'POST https://core.carefluence.com/cf.admin.core/connect/revocation'
---

# Authorize against Carefluence Open API R4

## Before you start

You need a registered developer account and an approved application. Carefluence
gates this: register at
`https://core.carefluence.com/cf.admin.core/Account/RegisterDeveloper`, sign in at
`https://core.carefluence.com/cf.admin.core/Account/Login`, register the app, and
wait for Carefluence administration to approve the grants. There is no way to get
a token without that approval, so do not design a flow that assumes self-service
issuance.

## Steps

1. **Discover, do not hard-code.** `GET {base}/.well-known/smart-configuration`
   where `{base}` is `https://classic.carefluence.com/r4/`. It is anonymous and
   returns `authorization_endpoint`, `token_endpoint`, `revocation_endpoint` and
   a `capabilities` array. Confirm `launch-standalone` (or `launch-ehr`) and
   `permission-patient` / `permission-user` are present for the flow you want.

2. **Read the full OpenID metadata** at
   `https://core.carefluence.com/cf.admin.core/.well-known/openid-configuration`
   for `introspection_endpoint`, `jwks_uri`, `code_challenge_methods_supported`
   and the complete `scopes_supported` list.

3. **Use authorization code with PKCE.** The provider states plainly that
   "the authorization code grant type is the only grant type supported in this
   release". The discovery document also advertises `implicit` and `password` —
   ignore them. `S256` is supported; use it.

   `GET {authorize}?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...&code_challenge=...&code_challenge_method=S256`

4. **Request only the scopes you will use.** Scopes are SMART v1 syntax and every
   clinical scope on this server is `.read` — there is no write scope advertised.
   See `scopes/carefluence-scopes.yml` for all 51. Add `offline_access` only if
   you genuinely need to act without the user present.

5. **Exchange the code** with `POST {token}` (form-encoded: `grant_type=authorization_code`,
   `code`, `redirect_uri`, `client_id`, `client_secret` for confidential clients,
   `code_verifier` for public clients). The published response shape is
   `{"access_token": "...", "expires_in": 1200, "token_type": "Bearer"}` — assume
   a short life and refresh rather than re-running the whole flow.

6. **Call resources** with `Authorization: Bearer <access_token>`.

7. **On 401**, the token is missing or invalid — refresh. **On 403**, the token is
   valid but the scope or the patient context does not cover the request; compare
   the granted scope against the resource you asked for rather than retrying.
   Errors come back as a FHIR `OperationOutcome`, not RFC 9457 problem+json — see
   `errors/carefluence-problem-types.yml`.

8. **Revoke** at `{revocation_endpoint}` when you are finished with a refresh
   token.

## Things that will catch you out

- Multi-factor authentication may be switched on for developers and patients
  (Carefluence SMART Connect). Interactive login can require an emailed or texted
  code; a headless flow that assumes a password-only login will hang.
- The authorization server lives on a different host (`core.carefluence.com`) and
  a different path prefix (`/cf.admin.core`) from the FHIR server
  (`classic.carefluence.com/r4`). Do not derive one from the other.
- `https://carefluence.com/.well-known/openid-configuration` is a 404. The
  marketing host serves no discovery documents at all.
