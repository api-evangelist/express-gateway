---
name: express-gateway-manage-scopes
description: >-
  Declare, inspect, grant and revoke Express Gateway scopes — the free-form
  permission tags that mark API endpoints and gate consumer credentials. Use when
  changing what an existing consumer is allowed to reach, or when setting up the
  permission vocabulary for a gateway.
api: express-gateway:express-gateway-scopes-api
generated: '2026-09-07'
method: generated
source: >-
  openapi/_original/express-gateway-openapi.yml (operationIds verified against the
  spec) + https://www.express-gateway.io/docs/credential-management/
operations:
  - listScopes
  - getScope
  - createScope
  - createScopes
  - deleteScope
  - setCredentialScopes
  - addCredentialScope
  - removeCredentialScope
  - listConsumerCredentials
---

# Manage Express Gateway scopes

## The model

Express Gateway does **not** define any scopes. There is no vendor catalog to look
up. A scope is a free-form tag the operator invents, declares on the gateway,
attaches to API endpoints, and grants to credentials. The same tag works across all
three credential types — `basic-auth`, `key-auth` and `oauth2` — so one vocabulary
governs key auth and OAuth 2.0 alike.

Two separate surfaces are involved, and confusing them is the usual mistake:

- **The gateway's scope registry** (`/scopes`) — which tags exist at all.
- **A credential's grants** (`/credentials/{type}/{id}/scopes`) — which tags one
  consumer holds.

Granting a scope that was never declared, or declaring one nobody was granted, both
silently do nothing useful.

## Declaring scopes

- **List what exists** — `listScopes`, `GET /scopes`.
- **Check one** — `getScope`, `GET /scopes/{scope}`. This is an existence check:
  200 means it exists, 404 means it does not. It is the only operation in the whole
  Admin API with a documented failure status, so use it rather than inferring.
- **Create one** — `createScope`, `PUT /scopes/{scope}`.
- **Create several** — `createScopes`, `POST /scopes`.

Always `getScope` before granting. A grant of an undeclared scope is the failure
mode that produces "the key works but the endpoint still rejects it".

## Granting and revoking

- **Replace the whole set** — `setCredentialScopes`,
  `PUT /credentials/{type}/{id}/scopes`. This is destructive: whatever the
  credential held before is gone. Read the current grants with
  `listConsumerCredentials` first.
- **Add exactly one** — `addCredentialScope`,
  `PUT /credentials/{type}/{id}/scopes/{scope}`.
- **Remove exactly one** — `removeCredentialScope`,
  `DELETE /credentials/{type}/{id}/scopes/{scope}`.

Prefer `addCredentialScope` / `removeCredentialScope` over `setCredentialScopes`
for routine changes. Add and remove are exact inverses on the same path, so a
mis-grant is fully undoable; a bad `set` has destroyed information you then have to
reconstruct.

## Deleting a scope

`deleteScope`, `DELETE /scopes/{scope}`, removes the tag from the gateway. It can be
recreated by name with `createScope` — but recreating it does **not** restore the
credential grants that referenced it. Those have to be re-granted one at a time.
Treat scope deletion as a change that needs the grant list captured first.

## Cautions

- **No idempotency.** No `Idempotency-Key` header exists on any operation. On a
  timeout, re-read with `getScope` or `listConsumerCredentials` before retrying.
- **No rate limits.** The Admin API imposes none and returns no rate-limit headers;
  it is a local administrative interface. Express Gateway's `rate-limit` policy
  applies to the operator's downstream APIs, not to this one.
- **No pagination you can drive.** `GET /scopes` and the other list operations
  return a `nextKey` field, but no request parameter to send it back is documented.
  On a large gateway, do not assume a single list call returned everything.
