# Incident Summary (Fill in)

**Title:** Task creation 500 errors and list performance degradation  
**Date:** 2026-06-09  
**Severity:** P2 — partial service degradation, ~35% of create-task requests failing

## Impact

- Customers intermittently unable to create tasks (~35% failure rate)
- Some users experiencing slow task list load times (up to 1847ms)
- Tasks appearing duplicated in UI after refresh

## Detection

- Customer reports via support tickets
- Log artifact (artifacts/sample_api_log.txt) showing unhandled FormatException on POST /api/tasks

## Timeline (UTC)

- 00:00 — Customer reports received: intermittent 500s on task creation, slow lists, duplicate tasks
- 00:15 — Log artifact reviewed, FormatException identified in sample_api_log.txt
- 00:30 — Root cause identified: missing X-Client-Timestamp header causing DateTime.Parse to throw
- 00:45 — Secondary issues identified: in-memory list filtering, concat bug in UI
- 01:00 — Fixes implemented and manually verified
- 01:30 — Tests written and committed

## Root cause

Initially considered the issue might be request body validation, but the structured log line showing X-Client-Timestamp present=False immediately before the stack trace ruled that out.

- **500 errors:** `DateTime.Parse(clientTimestamp)` called unconditionally even when `X-Client-Timestamp` header was absent. The UI randomly omits the header ~35% of requests.
- **Slow lists:** Full table scan loaded all users' tasks into memory before filtering in C#.
- **Duplicates:** UI used `state.tasks.concat(items)` on every refresh instead of replacing the list.

## Mitigation / resolution

- Fell back to `DateTime.UtcNow` when header is missing
- Pushed WHERE, ORDER BY, and LIMIT into the database query
- Replaced `concat` with direct assignment in `refresh()`

## Verification

- Manually verified task creation no longer 500s after fix
- Confirmed list SQL now includes WHERE and LIMIT clauses in logs (elapsedMs dropped from 1847ms to 9ms)
- Confirmed tasks no longer duplicate on refresh

## Follow-ups / action items

- [x] Fix UI to always send X-Client-Timestamp header consistently
- [ ] Add database index on UserId for further query performance
- [ ] Add monitoring/alerting on 5xx error rates for /api/tasks
