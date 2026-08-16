---
name: flume-health-govern-discovery-access
description: Grant, approve, extend and revoke time-boxed Flume Context Discovery sessions — the approver-gated access control that lets an AI CLI tool reach a customer's data estate, with an audit trail.
api: Flume Console API
base_url: https://console.flumehealth.com
generated: '2026-08-16'
method: generated
source: openapi/flume-health-console-api-openapi.yml
operations:
  - listDiscoveryApprovers
  - createDiscoveryApprover
  - getDiscoveryApprover
  - revokeDiscoveryApprover
  - listDiscoveryConnections
  - createDiscoveryConnection
  - getDiscoveryConnection
  - updateDiscoveryConnection
  - deleteDiscoveryConnection
  - testDiscoveryConnection
  - grantDiscoverySession
  - createDiscoverySession
  - listDiscoverySessions
  - getDiscoverySession
  - approveDiscoverySession
  - denyDiscoverySession
  - extendDiscoverySession
  - cancelDiscoverySession
  - revokeDiscoverySession
  - launchDiscoverySession
  - listDiscoveryExtractions
  - executeDiscoveryExtraction
  - getDiscoveryExtraction
---

# Govern Context Discovery access

Context Discovery is Flume's access-control surface for letting tooling — including AI CLI tools — reach a
customer's data estate. Access is a **session**: requested, approved by a named approver, time-boxed, revocable,
and audited. Twenty-three of the API's 153 operations are here, which tells you how much of Flume's design weight
sits on gating this rather than on the data path itself.

## Before you start

`Authorization: Bearer <token>` from `https://auth.flumehealth.com/oauth/token` (audience
`https://console.flumehealth.com/api`), plus `X-Flume-Account-ID: <accountId>` on every call.

## Set up who can approve

1. `listDiscoveryApprovers` — `GET /api/v1/context/discovery/approvers`.
2. `createDiscoveryApprover` — `POST /api/v1/context/discovery/approvers` — name an approver.
3. `getDiscoveryApprover` — `GET /api/v1/context/discovery/approvers/{id}`.
4. `revokeDiscoveryApprover` — `POST /api/v1/context/discovery/approvers/{id}:revoke` — when a person leaves.
   Revoking the approver does **not** revoke sessions they already approved; walk those separately (step 12).

## Set up what can be reached

5. `listDiscoveryConnections` — `GET /api/v1/context/discovery/connections`.
6. `createDiscoveryConnection` — `POST /api/v1/context/discovery/connections`.
7. `testDiscoveryConnection` — `POST /api/v1/context/discovery/connections/{id}:test` — **always test before you
   grant a session against it.** A broken connection surfaces as a failed extraction later, which is a worse place
   to find it.
8. `updateDiscoveryConnection` / `deleteDiscoveryConnection` — `PATCH` / `DELETE` on
   `/api/v1/context/discovery/connections/{id}`.

## Run the session lifecycle

9. **Request.** `createDiscoverySession` — `POST /api/v1/context/discovery/sessions` — with the connection,
   requester, scope and the grant type. `grantDiscoverySession` —
   `POST /api/v1/context/discovery/sessions:grant` — is the direct-grant path. One grant type in the contract is
   `subscription_access`, which is what covers AI CLI tool use.
10. **Decide.** `approveDiscoverySession` — `POST /api/v1/context/discovery/sessions/{id}:approve` — or
    `denyDiscoverySession` — `:deny`.
11. **Use.** `launchDiscoverySession` — `POST /api/v1/context/discovery/sessions/{id}/launches` — records a single
    AI CLI tool launch under the session. The contract states this writes to the audit log with
    `event=isolated_session_launch`. The session must exist, belong to the caller, be a `subscription_access`
    grant, be approved, and be neither expired nor revoked — five conditions, and a failure on any of them is a
    `4xx`, not a silent no-op.
12. **End it.** `extendDiscoverySession` (`:extend`) to push the expiry out, `cancelDiscoverySession`
    (`:cancel`) for a session that is no longer needed, `revokeDiscoverySession` (`:revoke`) to kill access now.
13. **Review.** `listDiscoverySessions` — `GET /api/v1/context/discovery/sessions` — filter on `status`,
    `requester` and `scope`. `getDiscoverySession` for one.

## Extract

14. `executeDiscoveryExtraction` — `POST /api/v1/context/discovery/extractions` — run an extraction under an
    approved session. `listDiscoveryExtractions` and `getDiscoveryExtraction` to follow it.

## Rules

- **Sessions expire; that is the feature.** Prefer `:extend` on a live session over creating a second one — a
  second session splits the audit trail for the same piece of work.
- **`:revoke` is immediate, `:cancel` is administrative.** Use `:revoke` for a suspected compromise.
- **Every launch must be recorded.** `launchDiscoverySession` is what puts the tool use in the audit log. An agent
  that reaches the data without calling it leaves no trail — in a HIPAA-adjacent estate that is the finding, not a
  detail.
- **No idempotency keys.** A retried `createDiscoverySession` creates a second session. List by `requester` and
  `status` before creating.
- **Errors** are `{ code, message, details[] }` as `application/json`, not RFC 9457. `403` means the caller is not
  scoped to this account or session; `409` means a state conflict (approving an already-decided session, extending
  a revoked one). Keep the `x-trace-id` response header.
