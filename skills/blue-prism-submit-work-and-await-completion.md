---
name: blue-prism-submit-work-and-await-completion
description: >-
  Submit a business item into a Blue Prism work queue and learn when a digital worker finishes it,
  using a webhook callback instead of polling.
api: Blue Prism API 7.5.1
generated: '2026-08-29'
method: generated
source: >-
  openapi/blue-prism-enterprise-api-openapi.yml and
  https://documentation.blueprism.com/bp-7-5/en-us/helpWebhooks.htm
operations:
  - getWorkqueues
  - getWorkQueue
  - createWorkQueueItem
  - CreateWebHookWorkQueueItem
  - getWorkQueueItemFromWorkQueue
  - createWorkQueueItemAttempt
---

# Submit work to Blue Prism and be told when it is done

## Before you start

- The base URL is **environment-specific**: `http(s)://<address>/api/v7`. Blue Prism Enterprise is
  installed on the customer's own server, so there is no shared host. Get `<address>` from the
  operator; do not guess it.
- Get a bearer token from the customer's Blue Prism Authentication Server using the
  **client_credentials** grant against `<auth-server>/connect/token`, scopes `bp-api bpserver`.
  Send it as `Authorization: Bearer <jwt>`.
- **Every** operation can return `401` (token missing/expired — re-acquire and retry) or `403`
  (the service account lacks the Blue Prism permission this endpoint needs — NOT retryable; the
  per-endpoint permission matrix is at
  <https://documentation.blueprism.com/bp-7-5/en-us/bp-api/api-user-permissions.htm>).

## Steps

1. **Find the queue.** `GET /api/v7/workqueues` (`getWorkqueues`) — or
   `GET /api/v7/workqueues/light` for a smaller payload. Collections are cursor-paged: pass
   `itemsPerPage` and follow `pagingToken` until it stops coming back. There is no total count.
2. **Read the queue's contract.** `GET /api/v7/workqueues/{workQueueId}` (`getWorkQueue`). Note
   `keyField` (the queue's business key), `isEncrypted`, and `maxAttempts` — `maxAttempts` is the
   ceiling on step 6.
3. **Submit the item.** `POST /api/v7/workqueues/{workQueueId}/items` (`createWorkQueueItem`), or
   `POST .../items/batch` (`createWorkQueueItems`) for many at once.
   **THERE IS NO IDEMPOTENCY KEY IN THIS API.** If the call times out you cannot safely repeat it.
   Recover by reading the queue back with `GET /api/v7/workqueues/{workQueueId}/items` and
   matching on your own `keyValue` before you consider re-posting.
4. **Subscribe to the outcome.**
   `POST /api/v7/workqueues/{workQueueId}/items/{workQueueItemId}/callbacks`
   (`CreateWebHookWorkQueueItem`). Required body fields:
   - `eventtypes` — choose from `completed`, `exception`, `prioritychanged`, `dataupdated`,
     `status`, `deferred`, `locked`, `retryexception`.
   - `callbackurl` — where Blue Prism POSTs the callback.
   - `extensions.security` — `credentialName` (a credential registered in the Blue Prism
     Credential Manager) and `type` (e.g. `BasicAuthentication`).

   Keep the `secret` from the response: it is what validates the callback signature. Subscriptions
   are **per item**, not per queue — there is no environment-wide event stream.
5. **Handle the callback.** A terminal event (`completed` / `exception`) deactivates the
   subscription once the callback succeeds *or* fails. From 7.5.1, repeatable events
   (`dataupdated`, `locked`, `retryexception`) leave the subscription active.
6. **On exception, retry deliberately.**
   `POST /api/v7/workqueues/{workQueueId}/items/{workQueueItemId}/attempts`
   (`createWorkQueueItemAttempt`) forces the item to retry. Read `attemptNumber` on the item first
   and stop at the queue's `maxAttempts`. **No retry window is published** — do not assume one.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 400 | `ValidationError` (`invalidField`, `message`) or `UrlParameterError` (`message`, `messageDetail`) | Fix the input. Not retryable as sent. |
| 401 | Token missing or expired | Re-acquire the JWT and retry once. |
| 403 | Permission denied | Not retryable. Escalate to the operator. |
| 404 | Queue or item not found | Verify the id. |
| 409 | Conflicts with current state | Re-read and reconcile. |

There is **no `429`** anywhere in this API and no rate-limit header. Throughput is bounded by the
customer's own server, so back off on latency, not on a signal.
