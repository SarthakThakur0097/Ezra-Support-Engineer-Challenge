# Follow-up Ticket

**Title:** Add database index on UserId and production monitoring for task API  
**Priority:** P2  
**Owner:** Engineering  

## Description
With the immediate incident resolved (500 errors fixed, list query optimized, UI header 
bug fixed), these follow-up improvements will make the service more resilient and easier 
to diagnose in future incidents.

## Acceptance criteria
- [ ] Alerting configured on 5xx error rate for POST `/api/tasks` — threshold recommended at >5% error rate over a 5-minute window
- [ ] `dotnet test` runs cleanly in an automated pipeline on every push

## Notes / context
- Relevant code: `src/SupportEngineerChallenge.Api/Data/AppDbContext.cs` — add index in `OnModelCreating`
- Monitor `ListTasks completed elapsedMs` in logs — if it climbs again as data grows, the index is needed
- Consider adding structured logging on 5xx responses to make future diagnosis faster
- Database index on `Tasks.UserId` has been implemented in `AppDbContext.cs` via `b.HasIndex(x => x.UserId)` in `OnModelCreating`