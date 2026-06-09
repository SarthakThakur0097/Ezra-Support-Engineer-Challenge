# Follow-up Ticket

**Title:** Add database index on UserId and production monitoring for task API  
**Priority:** P2  
**Owner:** Engineering  

## Description
With the immediate incident resolved (500 errors fixed, list query optimized, UI header 
bug fixed), these follow-up improvements will make the service more resilient and easier 
to diagnose in future incidents.

## Acceptance criteria
- [ ] Database index added on `Tasks.UserId` column
- [ ] Alerting configured on 5xx error rate for POST `/api/tasks`
- [ ] `dotnet test` passes in CI with no environment-specific workarounds

## Notes / context
- Relevant code: `src/SupportEngineerChallenge.Api/Data/AppDbContext.cs` — add index in `OnModelCreating`
- Monitor `ListTasks completed elapsedMs` in logs — if it climbs again as data grows, the index is needed
- Consider adding structured logging on 5xx responses to make future diagnosis faster