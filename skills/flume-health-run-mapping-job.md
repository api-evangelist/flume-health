---
name: flume-health-run-mapping-job
description: Build, auto-map, test and version a Flume field-mapping job — the v2 trades job surface plus Endpoint Maps, Shards and Source Files that turn a partner file into canonical Flume Data Model records.
api: Flume Console API
base_url: https://console.flumehealth.com
generated: '2026-08-16'
method: generated
source: openapi/flume-health-console-api-openapi.yml
operations:
  - listJobs
  - createJob
  - getJob
  - updateJob
  - testJobMap
  - createAutomapJob
  - listAutomapJobs
  - getAutomapJobsById
  - updateAutomapJob
  - getAutomapJobOutputs
  - listEndpointMapVersions
  - getEndpointMapVersion
  - testEndpointMap
  - listSourceFiles
  - searchSourceFiles
  - getSourceFile
  - listShards
  - getShard
---

# Build and run a mapping job

Flume's job surface is where a partner's raw file becomes canonical Flume Data Model records. It is the newest part
of the API — the jobs live under `/api/v2/trades/`, while everything they operate on (Endpoints, Maps, Shards,
Source Files) is still `/api/v1/`. Both versions are live on the same host at the same time.

## Before you start

`Authorization: Bearer <token>` from `https://auth.flumehealth.com/oauth/token` (audience
`https://console.flumehealth.com/api`), plus `X-Flume-Account-ID: <accountId>` on every call.

## Steps

1. **See what already exists.** `listJobs` — `GET /api/v2/trades/jobs`. Filter and page with `pageToken` /
   `pageSize`; sort with `orderBy` / `orderDesc`.

2. **Create the job.** `createJob` — `POST /api/v2/trades/jobs` — bound to the Endpoint whose data it maps.

3. **Let Flume propose the mapping.** `createAutomapJob` —
   `POST /api/v2/trades/jobs/{jobId}/automapjobs` — starts an automap run. Automap results are **versioned**:
   - `listAutomapJobs` — `GET /api/v2/trades/jobs/{jobId}/automapjobs`
   - `getAutomapJobsById` — `GET /api/v2/trades/jobs/{jobId}/automapjobs/{automapversion}`
   - `getAutomapJobOutputs` — `GET /api/v2/trades/jobs/{jobId}/automapjobs/{automapversion}/outputs` — the
     proposed field bindings. **Read these before accepting anything.**
   - `updateAutomapJob` — `PATCH /api/v2/trades/jobs/{jobId}/automapjobs/{automapversion}` — to correct them.

4. **Test the map against real data, twice.**
   - `testJobMap` — `POST /api/v2/trades/jobs/{id}/tests/map` — tests the job's map.
   - `testEndpointMap` — `POST /api/v1/endpoints/{id}/tests/map` — tests the map as the Endpoint will run it.
   Never promote a map you have not run through both.

5. **Check the map version history.** `listEndpointMapVersions` —
   `GET /api/v1/endpoints/{id}/maps/{fieldType}` — and `getEndpointMapVersion` —
   `GET /api/v1/endpoints/{id}/maps/{fieldType}/{ordinal}`. Maps are ordinal-versioned; the ordinal is how you
   compare what changed and how you roll back by re-pointing.

6. **Watch the input side.** `listSourceFiles` — `GET /api/v1/sourceFiles` — supports `isProcessed`,
   `isStaged`, `fileDisappeared` and `shardBatchKey` filters, which are the four questions you actually ask when a
   file did not land. `searchSourceFiles` — `GET /api/v1/sourceFiles:search` — for free-text. `getSourceFile` for
   one.

7. **Watch the output side.** `listShards` — `GET /api/v1/endpoints/{endpointId}/shards` — and `getShard` for a
   single shard. Shards are how a batch is split for processing; a stuck shard is where a stalled trade shows up.

8. **Update the job** with `updateJob` — `PATCH /api/v2/trades/jobs/{id}` — once the map is proven.

## Rules

- **Automap is a proposal, not a decision.** Always read `getAutomapJobOutputs` and correct with
  `updateAutomapJob` before running against production data. This is eligibility and claims data for real members.
- **Version, don't overwrite.** Map versions are ordinals under `/maps/{fieldType}/{ordinal}`. Preserve the prior
  ordinal so a bad map can be backed out.
- **No idempotency keys anywhere in this API.** A retried `createJob` or `createAutomapJob` can create a duplicate
  run. List first, then create.
- **Bulk source-file writes** (`createSourceFileBulk` / `updateSourceFilesBulk` at `/api/v1/sourceFiles:bulk`)
  return `413 Payload Too Large` when the body is oversized — one of only three operations that declare it. Chunk.
- **Errors** are `{ code, message, details[] }` as `application/json`, not RFC 9457. Keep the `x-trace-id` response
  header from any failure.
