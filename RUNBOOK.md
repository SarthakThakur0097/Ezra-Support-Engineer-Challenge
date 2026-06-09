# Runbook — SupportEngineerChallenge

## Service overview

- **Service:** SupportEngineerChallenge.Api
- **Purpose:** Minimal task tracker (create + list tasks)
- **Data store:** SQLite (`app.db` in the API working directory)

## Common commands

**Run locally**

```bash
cd src/SupportEngineerChallenge.Api
dotnet run
```

**Run tests**

```bash
dotnet test
```

## Key endpoints

- `GET /api/tasks?userId={id}&limit={n}`
- `POST /api/tasks`

## Using log artifacts

- **Create-task 500:** Inspect `artifacts/sample_api_log.txt` (or production logs). Look for the `CreateTask request` line — `X-Client-Timestamp present=False` or `length=0` indicates missing/invalid header. The stack trace shows `FormatException` at `DateTime.Parse` in `TaskEndpoints.cs`. This means the server received an empty timestamp and crashed trying to parse it.
- **Slow list:** Look for `ListTasks completed` lines with high `elapsedMs` (e.g. `artifacts/sample_slow_list_log.txt`). Correlate `userId` and `limit` with slow requests. High elapsedMs with large limits indicates a full table scan — check whether the SQL query logged by EF Core includes a WHERE clause.

## Troubleshooting checklist

### "Create task fails with 500"

1. Check API logs for `FormatException: String '' was not recognized as a valid DateTime`
2. Look for `X-Client-Timestamp present=False` in the `CreateTask request` log line above the exception
3. If present, the client is not sending the header — server must fall back to `DateTime.UtcNow`
4. Verify fix: `var createdAt = hasTimestamp ? DateTime.Parse(clientTimestamp) : DateTime.UtcNow;`
5. Test by POSTing to `/api/tasks` without the `X-Client-Timestamp` header — should return 201

### "Tasks list is slow"

1. Find `ListTasks completed` lines with high `elapsedMs` in logs
2. Note the `userId` and `limit` values — high limits correlate with slow responses
3. Check EF Core SQL logs — if no `WHERE` clause is present, the full table is being loaded into memory
4. Fix: push `Where()`, `OrderByDescending()`, and `Take()` into the EF Core query before `ToListAsync()`
5. Verify: SQL logs should show `WHERE "t"."UserId" = @__userId_0 ... LIMIT @__p_1`

### "Duplicates / wrong order after refresh"

1. Open browser dev tools and check the API response — confirm no duplicates in the response itself
2. If response is clean but UI shows duplicates, the bug is in the UI state update
3. Check `main.js` `refresh()` — if it uses `state.tasks.concat(items)` tasks accumulate on every refresh
4. Fix: replace with `state.tasks = items` to replace instead of append
5. Verify: refresh multiple times and confirm task count stays consistent

## Verification steps

- POST to `/api/tasks` without `X-Client-Timestamp` header — confirm 201 response
- Check SQL logs on GET `/api/tasks` — confirm WHERE and LIMIT present, elapsedMs < 50ms
- Refresh UI multiple times — confirm no duplicate tasks appear
- Run `dotnet test` — all tests should pass

## Rollback / mitigation

- Roll back to last known good commit via `git revert`
- If create-task 500s return: add temporary server-side guard to reject requests with invalid timestamps with a 400 rather than crashing with 500
- If list slowness returns: add a database index on `UserId` as a mitigation while fix is prepared
- Monitor 5xx rate on POST `/api/tasks` after any deployment
