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


## Reproduction steps

**500 on task creation:**
1. Run the app locally with `dotnet run`
2. Open the UI and click Add Task repeatedly
3. ~35% of attempts return a 500 error
4. Check logs — `X-Client-Timestamp present=False` appears before the FormatException

**Slow task list:**
1. Check `artifacts/sample_slow_list_log.txt`
2. Note `elapsedMs=1847` for a request with `limit=200`
3. Check EF Core SQL logs — no WHERE clause present, full table scan confirmed

**Duplicate tasks:**
1. Run the app and load the task list
2. Click Refresh multiple times
3. Tasks accumulate with each refresh
4. Check API response in browser dev tools — response is clean, confirming the bug is in the UI

## Root cause

- **500 errors:** `DateTime.Parse(clientTimestamp)` called unconditionally even when `X-Client-Timestamp` header was absent. The UI randomly omits the header ~35% of requests.
- **Slow lists:** Full table scan loaded all users' tasks into memory before filtering in C#.

- **Duplicates:** UI used `state.tasks.concat(items)` on every refresh instead of replacing the list.

- The duplicate tasks report was initially unclear — confirmed it was a UI issue and not an API issue by checking the API response directly in the browser dev tools, which showed no duplicates. This pointed to the UI state update logic in main.js where concat was accumulating tasks on every refresh.



## What was considered / ruled out

- Initially considered the 500 error might be a request body validation issue, but the structured log line showing `X-Client-Timestamp present=False` immediately before the stack trace ruled that out.
- For the slow list, considered the issue might be dataset size or SQLite limitations, but the EF Core SQL logs confirmed no WHERE clause was present — the query was loading all users' tasks before filtering.

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
