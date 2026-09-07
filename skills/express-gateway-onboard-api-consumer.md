---
name: express-gateway-onboard-api-consumer
description: >-
  Onboard a new API consumer onto a running Express Gateway instance — create the
  user, register their application, issue a key-auth credential, and grant the
  scopes that let it reach protected endpoints. Use when someone needs API access
  through an Express Gateway deployment.
api: express-gateway:express-gateway-users-api
generated: '2026-09-07'
method: generated
source: >-
  openapi/_original/express-gateway-openapi.yml (operationIds verified against the
  spec) + https://www.express-gateway.io/docs/admin/
operations:
  - createUser
  - createApp
  - createCredential
  - setCredentialScopes
  - listConsumerCredentials
  - getUser
---

# Onboard an API consumer

## Before you start

The Express Gateway Admin API is **not a hosted service**. It runs on the operator's
own machine and binds by default to `http://localhost:9876`. There is no vendor
base URL, no account, and no signup. Ask the operator for the Admin API URL before
doing anything — it is either that localhost default (reachable only from the
gateway host) or a hostname they put in front of it.

**Authentication.** On its default binding the Admin API is unauthenticated. If the
operator has secured it the documented way — fronting it with Express Gateway under
the `key-auth` policy — send:

```
Authorization: apikey {keyId}:{keySecret}
```

**There is no idempotency mechanism.** No `Idempotency-Key` header exists. If a call
times out, do **not** blindly retry a create — read back with `getUser` or
`listConsumerCredentials` first and only retry if the object is genuinely absent.
`username` is a unique identifier, so a repeated `createUser` will fail rather than
duplicate; `createCredential` has no such protection and *will* issue a second
credential.

## Steps

1. **Create the user** — `createUser`, `POST /users`.
   `username`, `firstname` and `lastname` are required; `email` and `redirectUri`
   are optional, and `redirectUri` matters only if this consumer will use OAuth 2.0.
   The response returns a UUID `id` alongside the username. Either value works
   wherever `{id}` appears later.

2. **Register the application** — `createApp`, `POST /apps`.
   Apps represent non-human consumers and always belong to a user. Skip this step
   if the credential should belong to the person rather than to a piece of
   software.

3. **Issue the credential** — `createCredential`, `POST /credentials`.
   Choose the type deliberately:
   - `key-auth` — a keyId/keySecret pair. A consumer may hold **many** of these,
     which is what makes key rotation possible.
   - `basic-auth` — username and password. **One per consumer, maximum.**
   - `oauth2` — client secret or user password. **One per consumer, maximum.**

   Because a consumer can hold only one `basic-auth` and one `oauth2` credential,
   check `listConsumerCredentials` (`GET /credentials/{consumerId}`) before issuing
   either of those types.

4. **Grant scopes** — `setCredentialScopes`, `PUT /credentials/{type}/{id}/scopes`.
   Scopes must already exist on the gateway; see the
   `express-gateway-manage-scopes` skill. This operation **replaces the entire
   scope set** on the credential — read the current set first if you are adding to
   it, or use `addCredentialScope` to add exactly one without disturbing the rest.

5. **Verify** — `getUser` and `listConsumerCredentials`.
   Confirm the consumer exists, is active, and carries the credential and scopes
   you intended before telling anyone they have access.

## Undoing this

Every step here is reversible **except deletion**:

- Deactivate the credential — `setCredentialStatus`, `PUT /credentials/{type}/{id}/status`.
- Deactivate the user — `setUserStatus`, `PUT /users/{id}/status` with `status: false`.
- Remove one scope — `removeCredentialScope`, `DELETE /credentials/{type}/{id}/scopes/{scope}`.

`deleteUser` and `deleteApp` return 204 and have **no** restore path. Reach for
deactivation, not deletion, unless removal is genuinely what was asked for.

## What is not documented

Failure responses. The Admin API Reference documents success bodies only and names
exactly one error status across the whole reference (a 404 on scope lookup). Treat
any non-2xx as opaque, log the body verbatim, and do not pattern-match on an error
shape — there isn't a published one.
