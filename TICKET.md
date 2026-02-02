# Follow-up Ticket

**Title:** Post-Incident Improvements — Task Tracker  
**Priority:** P2  
**Owner:** Kwabena Bawuah (Lead Support Engineer)

## Description

During incident resolution, we applied minimal safe fixes to restore service. This ticket tracks follow-up improvements identified during investigation.

---

## Issue #1: UI Random Header Behavior

**Status:** Not fixed (intentionally deferred)

The UI still randomly sends an empty `X-Client-Timestamp` header ~35% of the time. The server-side fix handles this gracefully, but the UI bug should be corrected.

**Location:** `main.js:59-66`

**Acceptance Criteria:**
- [ ] Remove random behavior — always send valid ISO timestamp
- [ ] Verify no more `Invalid or missing X-Client-Timestamp` warnings in logs

---

## Issue #2: Database Performance

**Status:** Fixed, but further optimization possible

Query filtering now happens at database level. Additional indexing would improve performance at scale.

**Location:** `TaskEndpoints.cs:13-21`, database schema

**Acceptance Criteria:**
- [ ] Add database index on `UserId` column
- [ ] Add database index on `CreatedAt` column
- [ ] Review other endpoints for similar LINQ-to-Objects patterns

---

## Issue #3: Frontend Test Coverage

**Status:** Fixed, but no automated tests

The duplicate/ordering bug is fixed, but there's no test infrastructure to prevent regression.

**Location:** `main.js:46-47`

**Acceptance Criteria:**
- [ ] Set up frontend test framework (Jest or similar)
- [ ] Add test: refresh should not create duplicates
- [ ] Add test: task order should be newest-first

---

## Notes / Context

**Why deferred:**
- Kept incident fixes minimal for production safety
- UI changes require separate deployment cycle
- Database schema changes need migration planning

**Monitoring:**
- Track `Invalid or missing X-Client-Timestamp` warning frequency
- Monitor `/api/tasks` response times for regression

**Related files:**
- `src/SupportEngineerChallenge.Api/wwwroot/main.js`
- `src/SupportEngineerChallenge.Api/Endpoints/TaskEndpoints.cs`
- `src/SupportEngineerChallenge.Api/Data/AppDbContext.cs`
