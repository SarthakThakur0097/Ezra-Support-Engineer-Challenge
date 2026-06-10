# Support Engineer Challenge — Debug & Stabilize

This repo contains a small prebuilt app with a few realistic "production" issues. Your goal is to **triage**, **diagnose**, and **ship safe fixes** with clear communication.

## Scenario

Assume this app is running in production and you're on-call. We’ve received customer reports:

1) “Sometimes creating a task fails with a 500.”
2) “The tasks list is slow for some users.”
3) “We’ve seen tasks appear duplicated or out of order after refresh.”
4) "We're seeing 500 errors on task creation in production but can't reproduce locally. A log snippet is in **artifacts/sample_api_log.txt** — use the logs to identify the cause and fix it."

Not all reports may be accurate — part of the exercise is determining what’s real, what’s reproducible, and what you’d do next.

## What’s here

- `src/SupportEngineerChallenge.Api` — .NET 8 minimal API + SQLite + static UI
- `tests/SupportEngineerChallenge.Tests` — xUnit tests (some may fail)
- `RUNBOOK.md` — **you will update**
- `INCIDENT.md` — **you will fill in**
- `TICKET.md` — **you will write a follow-up ticket**
- `artifacts/` — sample logs + incident report context

## Timebox

Aim for ~2–4 hours. If you go beyond that, please note it.

## Getting started

Requirements:
- .NET SDK 8.x

Run the API + UI:

```bash
cd src/SupportEngineerChallenge.Api
dotnet restore
dotnet run
```

Then open:
- API Swagger: http://localhost:5088/swagger
- UI:         http://localhost:5088/

Run tests:

```bash
dotnet test
```

## Deliverables

Please complete:

1. **Triage & reproduction**  
   - Use the provided log artifacts (e.g. `artifacts/sample_api_log.txt`) to diagnose at least one issue
   - Clear repro steps for each confirmed issue
   - Capture evidence (logs, stack traces, etc.)

2. **Root cause analysis**  
   - Explain what’s happening and why (briefly)

3. **Fixes**  
   - Safe, minimal fixes
   - Add/update tests where appropriate

4. **Operational thinking**  
   - Update `RUNBOOK.md` with diagnosis + verification + rollback/mitigation steps

5. **Incident summary**  
   - Fill in `INCIDENT.md` with impact/timeline/root cause/fix/follow-ups

6. **Follow-up ticket**  
   - Write `TICKET.md` with a high-quality ticket (acceptance criteria, priority, etc.)

## Notes

- You can change anything in this repo (including UI) as long as you explain your choices.
- If you get blocked by setup, write down what you tried and where you got stuck.



---

## Debug Findings & Analysis

### What I Found and Fixed

**1. Intermittent 500 on task creation**

Started with the log artifact. The stack trace pointed straight to `DateTime.Parse` throwing on an empty string, and the line above it showed `X-Client-Timestamp present=False` — which made the cause pretty clear. The server was trying to parse the header unconditionally even when if it wasn't there.

Digging into `main.js` revealed why it was intermittent — the UI was randomly omitting the header about 35% of the time via `Math.random()`.

Fixed the server to use `DateTime.TryParse` with a fallback to `DateTime.UtcNow` so it handles both missing and malformed headers gracefully. Also fixed the UI to always send the header consistently.

Added a new test `CreateTask_ShouldReturn201_WhenTimestampHeaderMissing` to cover this case going forward.

---

**2. Slow task list**

The slow list log showed `elapsedMs=1847` for a request with `limit=200`. Looking at the code, the endpoint was calling `ToListAsync()` on the full tasks table and then filtering in C# — so every request was loading every user's tasks into memory regardless of who was asking.

Moved the `Where()`, `OrderByDescending()`, and `Take()` into the database query before the `ToListAsync()` call. Response time dropped from ~1847ms to ~9ms and the SQL logs now show the WHERE and LIMIT clauses hitting the database directly.

Also added a `UserId` index in `AppDbContext.cs` to keep this query fast as the dataset grows.

---

**3. Duplicate tasks on refresh**

Reproduced this by clicking Refresh a few times and watching the task count grow. Checked the API response in browser dev tools first — the response itself was clean, so the bug was definitely in the UI.

Traced it to `state.tasks.concat(items)` in `main.js` which was appending the fetched tasks onto the existing list every time instead of replacing it. One line fix — replaced concat with direct assignment.

---

### Tradeoffs

- Kept all fixes minimal — no unnecessary refactoring beyond what was needed to resolve each issue
- Chose `DateTime.TryParse` over a simple null check because it also handles malformed timestamps, not just missing ones
- Fixed the UI header bug in addition to the server-side guard — the server fallback is a safety net but the client behavior was still wrong

