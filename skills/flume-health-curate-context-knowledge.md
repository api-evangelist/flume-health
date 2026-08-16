---
name: flume-health-curate-context-knowledge
description: Work the Flume Context knowledge graph — search it, propose knowledge, move items through the review queue, attach artifacts, and supersede or erase entries — the surface behind Flume's AI agents.
api: Flume Console API
base_url: https://console.flumehealth.com
generated: '2026-08-16'
method: generated
source: openapi/flume-health-console-api-openapi.yml
operations:
  - getGraphStatus
  - searchGraph
  - queryGraph
  - listKnowledge
  - getKnowledge
  - getFullEntity
  - getGraphPeekNode
  - createKnowledge
  - bulkWriteKnowledge
  - updateKnowledge
  - listKnowledgeProposals
  - getKnowledgeReviewQueue
  - getKnowledgePromotionPreflight
  - approveKnowledgeReviewItem
  - mergeKnowledgeReviewItem
  - rejectKnowledgeReviewItem
  - dismissKnowledgeReviewItem
  - createKnowledgeArtifact
  - getKnowledgeArtifactContent
  - linkKnowledgeArtifact
  - getAttachCandidates
  - attachKnowledge
  - detachKnowledge
  - supersedeKnowledge
  - eraseKnowledge
  - feedbackKnowledge
  - getTurnKnowledge
---

# Curate the Context knowledge graph

Flume's Context layer maps a customer's data estate into a knowledge graph, and that graph is what its actuarial,
claims, coding and compliance agents reason over. This skill is the human-and-agent workflow for keeping it
correct. It is also the REST surface that sits alongside Flume's OAuth-protected MCP endpoint at
`/api/v1/context/mcp` — same namespace, same host.

## Before you start

`Authorization: Bearer <token>` from `https://auth.flumehealth.com/oauth/token` (audience
`https://console.flumehealth.com/api`), plus `X-Flume-Account-ID: <accountId>` on every call.

## Read the graph

1. `getGraphStatus` — `GET /api/v1/context/graph` — is the graph built and current?
2. `searchGraph` — `GET /api/v1/context/graph:search` with `q` — free-text entry point.
3. `queryGraph` — `POST /api/v1/context/graph:query` — structured traversal.
4. `getKnowledgeGraph` — `GET /api/v1/context/knowledge/graph` — the knowledge-side projection.
5. For one entity: `getKnowledge` (`/{id}`), `getFullEntity` (`/{id}/full`) for everything, or `getGraphPeekNode`
   (`/{id}/peek`) for a cheap look before you pull the full record.
6. `getKnowledgePrefix` — `GET /api/v1/context/knowledge/prefix` — for prefix/typeahead resolution.

## Write knowledge

7. `createKnowledge` — `POST /api/v1/context/knowledge` — one entry.
   `bulkWriteKnowledge` — `POST /api/v1/context/knowledge:bulk` — many. **Bulk writes return per-item errors**, not
   a single status: each failed item carries a `knowledgemanager.KnowledgeWriteError` with `code`, `field` and
   `message`, where `code` is one of `invalid_input`, `disambiguation_required`, `conflict`, `scope_forbidden`,
   `not_found`, `internal`. Iterate the item errors — a `200` on the bulk call does not mean every item landed.
   `disambiguation_required` in particular means the graph found more than one candidate and needs you to choose,
   not that the write failed permanently.
8. `updateKnowledge` — `PATCH /api/v1/context/knowledge/{id}`.

## Work the review queue

9. `listKnowledgeProposals` — `GET /api/v1/context/knowledge/proposals` — what the system wants to add.
10. `getKnowledgeReviewQueue` — `GET /api/v1/context/knowledge/review-items` — what is waiting on a human.
11. `getKnowledgePromotionPreflight` —
    `GET /api/v1/context/knowledge/review-items/{review_item_id}/promotion-preflight` — **run this before you
    approve.** It tells you what promoting the item will do to the graph.
12. Then act, with the colon-verb custom methods on
    `/api/v1/context/knowledge/review-items/{review_item_id}`:
    `:approve`, `:merge` (fold into an existing entity), `:reject`, `:dismiss`.

## Attach evidence

13. `createKnowledgeArtifact` — `POST /api/v1/context/knowledge/artifacts` — store the source document.
14. `linkKnowledgeArtifact` — `POST /api/v1/context/knowledge/{id}:link-artifact` — bind it to the entity.
15. `getKnowledgeArtifactContent` — `GET /api/v1/context/knowledge/artifacts/{content_hash}/content` — artifacts
    are addressed by **content hash**, so the same bytes are the same artifact. Compute the hash rather than
    re-uploading.

## Restructure

16. `getAttachCandidates` — `GET /api/v1/context/knowledge/{id}/attach-candidates` — read this first.
17. `attachKnowledge` / `detachKnowledge` — `POST /api/v1/context/knowledge/{id}:attach` / `:detach`.
18. `supersedeKnowledge` — `POST /api/v1/context/knowledge/{id}:supersede` — **the correct way to replace a wrong
    fact.** It preserves lineage.
19. `eraseKnowledge` — `POST /api/v1/context/knowledge/{id}:erase` — destructive, no lineage. Use it only when the
    content must not exist (for example PHI that should never have been ingested), never for ordinary corrections.

## Close the loop

20. `feedbackKnowledge` — `POST /api/v1/context/knowledge/{id}:feedback` — record whether an answer was right;
    `listKnowledgeFeedback` — `GET /api/v1/context/knowledge/{id}/feedback` — read it back.
21. `getTurnKnowledge` — `GET /api/v1/context/knowledge/turns/{message_id}` — trace which knowledge a specific
    agent turn used. This is your audit trail when an agent answer is challenged.

## Rules

- **Supersede, do not erase.** `:supersede` keeps the chain of what was believed and when; `:erase` destroys it.
  In a HIPAA-adjacent system the lineage is usually the point.
- **Preflight before promote.** `getKnowledgePromotionPreflight` exists because approving a review item mutates the
  graph the agents reason over.
- **Bulk is per-item.** Always walk `knowledgemanager.BulkItemError` results.
- **No idempotency keys.** Re-posting `createKnowledge` after a timeout can duplicate an entity; search first.
- **Errors** are `{ code, message, details[] }` as `application/json`, not RFC 9457. `403` here usually means the
  account in `X-Flume-Account-ID` is not scoped to that knowledge — the domain code `scope_forbidden` says the
  same thing inside a bulk result.
