---
name: blue-prism-run-a-process-and-read-its-logs
description: >-
  Start a Blue Prism automation on a runtime resource, watch it to completion, and read the
  per-stage execution log — including the licence check that must come first.
api: Blue Prism API 7.5.1
generated: '2026-08-29'
method: generated
source: openapi/blue-prism-enterprise-api-openapi.yml
operations:
  - getLicenseLimits
  - GetProcesses
  - getResources
  - createSession
  - updateSessionStartupParameters
  - GetSessions
  - getSession
  - getSessionLogs
  - deleteSession
---

# Run a Blue Prism process and read its logs

## Before you start

Base URL `http(s)://<address>/api/v7`; bearer JWT from the customer's Authentication Server
(client_credentials, scopes `bp-api bpserver`). See
[blue-prism-submit-work-and-await-completion](blue-prism-submit-work-and-await-completion.md) for
the auth and error rules, which apply identically here.

## Steps

1. **Check licence headroom FIRST.** `GET /api/v7/dashboards/currentLimitsAndUsage`
   (`getLicenseLimits`). Sessions are licence-limited. If you skip this, `POST /api/v7/sessions`
   returns `403` — and that same `403` is also what a permissions failure looks like, so you will
   not be able to tell the two apart from the status code alone. Read `errorMessage` on the
   `Error` body to disambiguate.
2. **Find the process.** `GET /api/v7/processes` (`GetProcesses`). Filter with `processName`,
   `processType`, `attributesInclude` / `attributesExclude`. Keep `processId`.
3. **Find a runtime resource.** `GET /api/v7/resources` (`getResources`). Prefer one whose
   `displayStatus` is idle and whose `activeSessionCount` leaves room; `poolId` tells you whether
   it is pooled. `GET /api/v7/resources/pools` (`getResourcePools`) lists pools.
4. **Create the session.** `POST /api/v7/sessions` (`createSession`) with `processId` and
   `resourceId`. Keep the returned `sessionId`.
   No idempotency key exists — if this times out, call `GET /api/v7/sessions` and look for a
   session matching your process/resource before creating another.
5. **Set startup parameters if the process needs them.**
   `PUT /api/v7/sessions/{sessionId}/parameters` (`updateSessionStartupParameters`). Confirm with
   `GET /api/v7/sessions/{sessionId}/parameters`. `areStartupParamsSet` on the session summary
   tells you whether this step is still outstanding.
6. **Watch it.** Poll `GET /api/v7/sessions/{sessionId}` (`getSession`) on `status`. There is no
   event stream for sessions — webhooks in this API cover work queue items only, so polling is
   the only option here. Use a backoff; there is no published rate limit to key on.
7. **Read the log.** `GET /api/v7/sessions/{sessionId}/logs` (`getSessionLogs`) for the full
   per-stage log, or `/logslight` (`getSessionLogsLight`) for a lighter payload. Filter with
   `stageName`, `stageType`, `logNumber`, `result`, `resultType`. Where `hasParameters` is true,
   `GET /api/v7/sessions/{sessionId}/logs/{logId}/parameters` returns the stage's inputs/outputs.
8. **On failure**, read `exceptionMessage`, `exceptionType`, `terminationReason` and `latestStage`
   from the session summary — they name the stage that broke, which is what you need to route the
   problem.

## Reversibility — read this before step 4

`DELETE /api/v7/sessions/{sessionId}` (`deleteSession`) deletes a **pending** session only. Once a
session has started, this API gives you no way to stop it — only a *schedule* has a stop
(`DELETE /api/v7/schedules/{scheduleId}/sessions`). And a stop is a cancel, not a rollback: work
the digital worker already performed inside the target application is not undone by anything in
this API. Treat step 4 as a commit point.
