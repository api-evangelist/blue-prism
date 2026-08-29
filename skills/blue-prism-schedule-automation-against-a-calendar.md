---
name: blue-prism-schedule-automation-against-a-calendar
description: >-
  Build a Blue Prism schedule with a working calendar, chain its tasks by success and failure, and
  trigger or stop it — plus the dependency check to run before deleting any of it.
api: Blue Prism API 7.5.1
generated: '2026-08-29'
method: generated
source: >-
  openapi/blue-prism-enterprise-api-openapi.yml and
  https://documentation.blueprism.com/bp-7-5/en-us/bp-api/api-example-usage.htm
operations:
  - getHolidayRegions
  - getHolidayByRegionId
  - createSchedule
  - createScheduleTask
  - createScheduleSessionForTask
  - updateStartupParameters
  - startSchedule
  - stopSchedule
  - getLogsForSchedule
  - getWorkQueueReferences
---

# Schedule a Blue Prism automation against a working calendar

## 1. Build the calendar

Blue Prism's own documented sequence
(<https://documentation.blueprism.com/bp-7-5/en-us/bp-api/api-example-usage.htm>):

1. `GET /api/v7/holidayRegions` — note the `regionId` you want.
2. `GET /api/v7/holidayRegions/{holidayRegionId}/publicHolidays` — note the ids of any public
   holidays you intend to **disable** for this calendar.
3. `POST /api/v7/calendars` with `name`, `workingWeek`, and
   `region: { regionId, disabledPublicHolidaysIds: [...] }`. Returns `calendarId`.
4. `POST /api/v7/calendars/{calendarId}/otherHolidays/batch` with
   `{ "holidays": ["2026-08-23", "2026-11-28"] }` to add non-public closures.

## 2. Build the schedule

- `POST /api/v7/schedules` (`createSchedule`) with the recurrence (`intervalType`, `timePeriod`,
  `startPoint` / `endPoint`, `dayOfWeek` / `dayOfMonth`, `startDate` / `endDate`) and the
  `calendarId` from step 1.
- `POST /api/v7/schedules/{scheduleId}/tasks` (`createScheduleTask`) for each step. The chain is
  explicit: `onSuccessTaskId` and `onFailureTaskId` point at the next task in each direction, and
  `failFastOnError` decides whether the schedule aborts. The schedule's `initialTaskId` is the
  entry point.
- `POST /api/v7/schedules/{scheduleId}/tasks/{taskId}/sessions`
  (`createScheduleSessionForTask`) attaches the process/resource pair a task runs.
- `PUT /api/v7/schedules/{scheduleId}/tasks/{taskId}/scheduledSessionParameters`
  (`updateStartupParameters`) sets that session's startup parameters.

## 3. Run and stop it

- `POST /api/v7/schedules/{scheduleId}/sessions` (`startSchedule`) triggers the schedule now.
- `DELETE /api/v7/schedules/{scheduleId}/sessions` (`stopSchedule`) requests it to stop. This is a
  **cancel**, not a rollback — anything the digital workers already did is not undone.
- `GET /api/v7/schedules/{scheduleId}/logs` (`getLogsForSchedule`) and
  `GET /api/v7/scheduleLogs/{scheduleId}` return execution history with `status`, `duration` and
  `serverName`.
- `POST /api/v7/schedules/{scheduleId}/clones` (`CloneSchedule`) copies a schedule rather than
  re-authoring it.

## 4. Before you delete anything

`deleteSchedule`, `DELETE /api/v7/calendars/{calendarId}`, `deleteWorkQueue`, `deleteWorkQueueGroup`,
`deleteEnvironmentVariable` and `deleteScheduleTask` have **no restore path in this API**. There is
no undelete, no trash, no restore window.

Three entities expose a dependency check — use it as your dry run:

- `GET /api/v7/calendars/{calendarId}/references`
- `GET /api/v7/workqueues/{workQueueId}/references`
- `GET /api/v7/environmentvariables/{environmentVariableId}/references`

A non-empty references list means something in the environment depends on the object. Retire
rather than delete where you can — `ScheduleSummary.isRetired` exists for exactly this.

## Errors

Same table as [blue-prism-submit-work-and-await-completion](blue-prism-submit-work-and-await-completion.md).
`409 Conflict` is common here: a duplicate schedule or calendar name, or a state transition that
is not legal from where the schedule currently is.
