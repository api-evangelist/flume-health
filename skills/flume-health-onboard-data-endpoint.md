---
name: flume-health-onboard-data-endpoint
description: Stand up a new source or destination Endpoint on the Flume Console API — create it, attach the right protocol secret, test the connection, and confirm it is exchanging data.
api: Flume Console API
base_url: https://console.flumehealth.com
generated: '2026-08-16'
method: generated
source: openapi/flume-health-console-api-openapi.yml
operations:
  - listAccounts
  - listConnections
  - createConnection
  - getEndpointDataTypes
  - createEndpoint
  - getEndpointSftpSecret
  - getEndpointCloudStorageSecret
  - getEndpointDatabaseSecret
  - getEndpointSnowflakeSecret
  - getEndpointApiSecret
  - testEndpointSftpConnection
  - postTestCloudStorageConnection
  - postTestDatabaseConnection
  - testEndpointSnowflakeConnection
  - getEndpoint
  - listTransactions
---

# Onboard a data Endpoint

In Flume, an **Endpoint** is a data source *or* destination for a single data type (eligibility, claims, and so on).
It syncs with the Flume Data Model over its own transmission protocol — SFTP, cloud storage, a database, Snowflake,
the Flume Lakehouse, or an API. This skill takes a new trading partner from nothing to a tested, live Endpoint.

## Before you start

Every call needs two things:

1. `Authorization: Bearer <token>` — get one from `https://auth.flumehealth.com/oauth/token` for audience
   `https://console.flumehealth.com/api`.
2. `X-Flume-Account-ID: <accountId>` — the header that selects which health-plan account you are operating on. It
   is required on 122 of the 153 operations in this API. Omitting it is the single most common failure.

## Steps

1. **Find the account.** `listAccounts` — `GET /api/v1/accounts`. Page with `pageSize` and follow `nextPageToken`
   until it is empty. Take the account id you will put in `X-Flume-Account-ID` for everything below.

2. **Find or create the Connection.** A Connection is the trading-partner relationship the Endpoint hangs off.
   `listConnections` — `GET /api/v1/connections` — and if the partner is new, `createConnection` —
   `POST /api/v1/connections`.

3. **Pick the data type.** `getEndpointDataTypes` — `GET /api/v1/endpoints/datatypes` — returns the datatypes Flume
   supports. `getEndpointDataType` — `GET /api/v1/endpoints/datatypes/{datatype}` — gives the field detail for one.
   Do not guess a datatype string; read it from this list.

4. **Create the Endpoint.** `createEndpoint` — `POST /api/v1/endpoints` — with the connection id, the datatype, the
   direction (source or destination), and the transmission protocol.

5. **Attach credentials for the protocol you chose.** Each protocol has its own secret resource under the Endpoint:
   - SFTP: `getEndpointSftpSecret` — `/api/v1/endpoints/{id}/secrets/sftp` (and
     `getEndpointSftpEncryptionSecret` at `/secrets/sftp/encryption` when PGP is in play)
   - Cloud storage: `getEndpointCloudStorageSecret` — `/api/v1/endpoints/{id}/secrets/cloudstorage` (plus
     `/secrets/cloudstorage/encryption`)
   - Database: `getEndpointDatabaseSecret` — `/api/v1/endpoints/{id}/secrets/database`
   - Snowflake: `getEndpointSnowflakeSecret` — `/api/v1/endpoints/{id}/secrets/snowflake`
   - Flume Lakehouse: `getEndpointFlumeLakehouseSecret` — `/api/v1/endpoints/{id}/secrets/flumelakehouse`
   - API: `getEndpointApiSecret` — `/api/v1/endpoints/{id}/secrets/api`

6. **Test the connection before you trust it.** Each protocol has a paired test operation:
   `testEndpointSftpConnection`, `postTestCloudStorageConnection`, `postTestDatabaseConnection`,
   `testEndpointSnowflakeConnection` — all `POST /api/v1/endpoints/{id}/tests/<protocol>`. The matching `GET` on the
   same path (`getEndpointSftpConnection`, `getTestCloudStorageConnection`, `getTestDatabaseConnection`,
   `getEndpointSnowflakeConnection`) reads back the last result. Do not move on until the test passes.

7. **Confirm.** `getEndpoint` — `GET /api/v1/endpoints/{id}` — then `listTransactions` —
   `GET /api/v1/endpoints/{endpointId}/transactions` — to see data actually moving.

## Rules

- **No idempotency keys.** This API declares no `Idempotency-Key` header on any operation. A retried
  `createEndpoint` or `createConnection` may create a duplicate. Read back with a list operation before retrying a
  create, never blind-retry.
- **Pagination is cursor-based.** `pageToken` + `pageSize` in, `nextPageToken` out. An empty or absent
  `nextPageToken` is the last page.
- **Errors** are the vendor envelope `{ code, message, details[] }` as `application/json` — *not* RFC 9457
  problem+json. On `400` read `details[]`; on `503` (declared on 18 operations, usually a dependency like SFTP or
  the warehouse being unreachable) back off and retry. Capture the `x-trace-id` response header on any failure —
  it is what Flume support will ask for.
- **Never log a secret response.** The `/secrets/*` operations return live trading-partner credentials.
